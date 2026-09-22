# 新机器人接入 rl_sar MuJoCo 仿真 · 移植手册

> 目标：让 `./cmake_build/bin/rl_sim_mujoco <robot> <scene>` 能跑起来并执行策略。
> 适用范围：**只覆盖 MuJoCo 仿真路径**。ROS1/ROS2/Gazebo/真机路径的差异见附录 A。
>
> 本文以把 `bpx` 接入的过程为原型写成，所有步骤都用一个占位符 `<robot>` 表示机器人名。
> 落地的实际值见附录 B，可直接对照。

---

## 0. 核心认知

整个移植过程之所以繁琐，是因为 **`rl_sim_mujoco` 和模型之间不是靠"名字"对接的，而是靠"下标"对接的**：

- 读关节状态：`mj_data->sensordata[固定下标]`
- 写电机指令：`mj_data->ctrl[固定下标]`

代码里**没有任何一处**去查找 `"FL_hip_joint"` 这个名字。所以 MJCF 里关节叫什么名字无所谓，**顺序**才是唯一重要的事。

理解这一点之后，下面所有步骤都只是"把顺序摆对"的体力活。

### 单一事实来源

出问题时永远回到这两个文件：

| 文件 | 定义了 |
|---|---|
| [`src/rl_sar/src/rl_sim_mujoco.cpp`](../src/rl_sar/src/rl_sim_mujoco.cpp) | 模型路径、传感器/执行器下标、PD 控制律 |
| [`src/rl_sar/library/core/rl_sdk/rl_sdk.cpp`](../src/rl_sar/library/core/rl_sdk/rl_sdk.cpp) | YAML 读取、观测拼接、动作→力矩换算 |

> 全文行号基于当前版本，改版后请以代码为准。

---

## 1. 六条硬契约

这六条是**不可协商**的接口。任何一条不满足，仿真要么崩溃、要么静默出错（机器人瘫软但不报错，最难查）。

### 1.1 路径契约

```cpp
// rl_sim_mujoco.cpp:64
std::string filename = std::string(CMAKE_CURRENT_SOURCE_DIR) + "/../rl_sar_zoo/"
                     + this->robot_name + "_description/mjcf/" + this->scene_name + ".xml";
```

`CMAKE_CURRENT_SOURCE_DIR` = `src/rl_sar`，所以实际路径：

```
src/rl_sar_zoo/<robot>_description/mjcf/<scene>.xml
```

**推论**：
- 描述目录必须叫 `<robot>_description`，与命令行参数强绑定，不能改名。
- `scene` 是**文件名**不是目录，可以随便取（如 `scene`、`scene_flat`），只要存在 `mjcf/<scene>.xml`。

### 1.2 传感器布局契约（★最容易错）

设 `N = num_of_dofs`。`GetState()` 按**标量偏移**硬编码读取：

| `sensordata` 标量偏移 | 内容 | 传感器元素 |
|---|---|---|
| `[0, N)` | 关节位置 | `<jointpos>` × N |
| `[N, 2N)` | 关节速度 | `<jointvel>` × N |
| `[2N, 3N)` | 关节力矩 | `<jointactuatorfrc>` × N |
| `[3N, 3N+4)` | IMU 姿态四元数 `(w,x,y,z)` | `<framequat>` × 1 |
| `[3N+4, 3N+7)` | IMU 角速度 | `<gyro>` × 1 |
| `[3N+7, ∞)` | **代码不读，随便放** | 任意 |

**最小 `nsensordata = 3N + 7`**。

#### ⚠️ 坑：传感器元素序号 ≠ sensordata 偏移

MuJoCo 里这是两个不同的量：

```
m.nsensor       // <sensor> 元素个数          —— 关节量每个元素占 1 个，IMU 四元数占 1 个元素但 4 个标量
m.nsensordata   // 标量总数                   —— 代码读的是这个缓冲
m.sensor_adr[i] // 第 i 个元素的标量起始偏移  —— 排布时要对齐的是它
m.sensor_dim[i] // 第 i 个元素占几个标量      —— 四元数=4，gyro/acc/framepos=3，关节量=1
```

> **写校验脚本时注意**：只需要检查每个传感器**段的起始 adr**（本节的 `0`、`N`、`2N`、`3N`、`3N+4`）。
> 不要去枚举区间内的每一个标量偏移——`body_quat` 占 `adr 36..39`，中间的 37/38/39 不在任何传感器的**起始** adr 上，
> 只按起始 adr 建表会误报它们"缺失"。若确实要展开区间，用 `sensor_adr[i] + k`（`k < sensor_dim[i]`）。

举例（N=12）：

```
元素 36  adr 36  body_quat   (占 adr 36..39)
元素 37  adr 40  body_gyro   (占 adr 40..42)   ← 元素序号 37，但偏移是 40
元素 38  adr 43  body_acc
元素 39  adr 46  body_pos
```

**按元素序号去数就会从 body_gyro 开始全错。** 校验时必须用 `m.sensor_adr`，见 §3 脚本一。

#### 坑：布局不对是**越界读**，不是"读错值"

如果传感器少了（比如 `nsensordata` 只有 37 而代码读到 42），`mj_data->sensordata[40]` 读的是**数组外的内存**——未定义行为。可能崩溃、可能读到垃圾值、也可能碰巧没事。所以这一步必须严格验证，不能"看着能跑就算了"。

### 1.3 执行器契约

```cpp
// rl_sim_mujoco.cpp SetCommand()
mj_data->ctrl[joint_mapping[i]] =
      motor_command.tau[i]
    + motor_command.kp[i] * (motor_command.q[i]     - sensordata[joint_mapping[i]])
    + motor_command.kd[i] * (motor_command.dq[i]    - sensordata[joint_mapping[i] + N]);
```

**PD 是 C++ 侧算的，写进 `ctrl` 的是力矩。** 因此：

- 执行器**必须是 `<motor>`**（力矩执行器），不能是 `<position>` / `<velocity>`。
- 策略输出的是**位置目标**（`action_scale * action + default_dof_pos`），不是力矩。训练时必须用同样的约定。

```xml
<actuator>
  <motor name="..." joint="..." gear="1" ctrllimited="true" ctrlrange="-30 30" />
  <!-- 12 个，顺序必须与 <sensor> 的关节顺序完全一致 -->
</actuator>
```

`ctrlrange` 要和 YAML 的 `torque_limits` 一致（或更宽）。

### 1.4 `joint_mapping` 契约

**定义（唯一）：`joint_mapping[i]` = 策略第 `i` 个关节 在 mjcf 中的下标。**

这一个数组同时用于 4 处，所以只要它对了就全部对齐：

```
GetState:      motor_state.q[i]  = sensordata[joint_mapping[i] + 0*N]
               motor_state.dq[i] = sensordata[joint_mapping[i] + 1*N]
               motor_state.tau_est[i] = sensordata[joint_mapping[i] + 2*N]
SetCommand:    ctrl[joint_mapping[i]] = ...
```

**推论**：`motor_state` / `motor_command` / 所有 YAML 里的 12 维向量，全都是**策略顺序**（= 训练顺序）。

**简化技巧**：把 `base.yaml` 的向量顺序**故意写成与 mjcf 完全一致**，则 `base.yaml` 里 `joint_mapping: [0,1,...,N-1]` 恒等；训练顺序与 mjcf 的差异全部收敛到 **策略 config.yaml 的 `joint_mapping`** 一处。官方 go2 就是这么分层的。

> 这里成立的前提是：`<actuator>` 的顺序与 `<jointpos>` 的顺序一致。
> 若不一致，需自行建立"策略序号 → actuator 序号"的独立映射。

> ⚠️ **`base.yaml` 恒等 ≠ 策略 config.yaml 也恒等。** 恰恰相反：config.yaml 的
> `joint_mapping` 基本不可能是恒等，因为训练侧的关节顺序由训练框架决定，与 mjcf 无关。
> 见 1.4.1。

#### 1.4.1 训练侧的关节顺序怎么定（★实测最常翻车的一步）

`joint_mapping` 和 `default_dof_pos` **必须同时从训练环境的 `joint_names` 推出来**，
分开猜必然对不上。三步：

**第一步 · 拿 ground truth（唯一权威来源）**

```python
# IsaacLab：env 建好之后
print(env.unwrapped.scene["robot"].joint_names)
```

这个列表就是**策略顺序**。特别注意它**不等于**：

- URDF 里 `<joint>` 的书写顺序 ❌
- mjcf 的 `<actuator>` / `<sensor>` 顺序 ❌

**第二步 · 换算 `joint_mapping`**

`joint_mapping[i]` = 策略第 `i` 个关节在 mjcf 里的下标：

```python
mjcf = ["fl_hip_roll_joint", "fl_hip_pitch_joint", "fl_knee_joint",     # bpx 的 mjcf 顺序
        "fr_hip_roll_joint", "fr_hip_pitch_joint", "fr_knee_joint",
        "hl_hip_roll_joint", "hl_hip_pitch_joint", "hl_knee_joint",
        "hr_hip_roll_joint", "hr_hip_pitch_joint", "hr_knee_joint"]
print([mjcf.index(n) for n in env.unwrapped.scene["robot"].joint_names])
```

**第三步 · 换算 `default_dof_pos`**

`default_dof_pos[i]` = `joint_names[i]` 这个关节在训练里的默认角，取自训练资产 cfg 的
`init_state.joint_pos` 正则（himloco 的 bpx 是 `.*_hip_roll=0.0` / `.*_hip_pitch=0.8` / `.*_knee=-1.5`）：

```python
def default_of(n):
    if n.endswith("hip_roll_joint"):  return 0.0
    if n.endswith("hip_pitch_joint"): return 0.8
    if n.endswith("knee_joint"):      return -1.5
print([default_of(n) for n in env.unwrapped.scene["robot"].joint_names])
```

> **一句判据**：`default_dof_pos[i]` 恒等于 `joint_names[i]` 这个关节的训练默认角。
> 两者同源，改一个必须改另一个。

##### 坑：IsaacSim 的 URDF importer **不保序**

IsaacLab 从 URDF 导入时，articulation 的关节顺序**不等于** URDF 里 `<joint>` 的书写顺序。
bpx 实测是**层序（宽度优先）**：

```
torso 的 4 个直接子关节 → 四条腿的 hip_roll
（腿的顺序 = URDF 里 leg link 的声明顺序 fl, fr, hl, hr）
        ↓
四个 hip_pitch  →  四个 knee
```

即 `fl_roll, fr_roll, hl_roll, hr_roll, fl_pitch, fr_pitch, hl_pitch, hr_pitch, fl_knee, fr_knee, hl_knee, hr_knee`，
对应 `joint_mapping: [0, 3, 6, 9,  1, 4, 7, 10,  2, 5, 8, 11]`。

而 bpx 的 URDF 里关节是按「每条腿一组」写的（`fl_roll, fl_pitch, fl_knee, fr_roll, …`），
**照抄文档顺序会得到恒等映射，那是错的**。

> 这条结论来自 bpx 一次实跑（见附录 B）。换机器人时仍要用第一步的 `joint_names` 确认，
> 不要直接套用"层序"这个经验。

##### 怎么验收（两个测试，各验一半）

站立姿态天生四腿对称（四条腿的 roll/pitch/knee 相同），所以腿序错了在站立时**看不出来**：

| 测试 | 验的是 | 通过 | 不通过 |
|---|---|---|---|
| `0` 站好后按 `1`，**不给速度指令** | 同一条腿内 roll/pitch/knee 的次序 | 原地站住 | 侧翻 / 肚皮朝上 |
| 按 `W` 给正前方指令 | 腿与腿之间的次序 | 直着往前走 | 横走 / 镜像 / 打转 |

**为什么"侧翻"指向腿内次序错**：站立时观测里 `dof_pos - default_dof_pos` 应该恰好是 0。
若只有腿序错，四条腿的值相同 → 差仍是 0 → 策略不会乱动；若腿内次序错，差会变成
`[±0.8, ∓0.8, 0, …]` 这种大值 → 策略立刻输出大动作 → roll 方向翻过去。

### 1.5 YAML 顶层 key 契约（★最容易错）

`ReadYaml` 做的是 `YAML::LoadFile(path)[file_path]`：

| 文件 | 路径 | 顶层 key 必须等于 |
|---|---|---|
| `policy/<robot>/base.yaml` | `<repo>/policy/<robot>/base.yaml` | `<robot>` |
| `policy/<robot>/<cfg>/config.yaml` | `<repo>/policy/<robot>/<cfg>/config.yaml` | `<robot>/<cfg>` |

**config.yaml 的 key 是完整相对路径，不是文件夹名。** 例如 go2 里写的是 `go2/himloco:`，不是 `himloco:`。

`<repo>` 由 CMake 定义：`POLICY_DIR = ${PROJECT_ROOT_DIR}/policy`，`PROJECT_ROOT_DIR` = 仓库根目录（`src/rl_sar/../..`）。**注意 `policy/` 在仓库根，不在 `src/` 下。**

#### 坑：YAML 读不到是**静默失败**

```cpp
// rl_sdk.cpp ReadYaml()
catch (const YAML::BadFile &e) {
    std::cout << LOGGER::ERROR << "..." << std::endl;
    return;              // ← 不抛异常！params 保持为空
}
```

`YamlParams::Get<T>(key)` 在 key 缺失时返回 `T()`（默认值），也不抛异常。所以：

- **`base.yaml` 不存在** → `num_of_dofs = 0`、`dt = 0.0f` → `InitJointNum(0)`，控制周期 0（CPU 忙循环），机器人完全瘫软且**不报错**。
- **config.yaml 顶层 key 写错** → `model_name = ""` → 模型路径退化成以 `/` 结尾。

**诊断技巧**：故意让模型文件缺失，看报错里打印的路径。若结尾是 `.../<cfg>/policy.pt` 说明 key 对了；若是 `.../<cfg>/`（一个斜杠结尾）说明 key 写错了。

**合并语义**：`base.yaml` 先加载，策略 `config.yaml` 的**同名 key 覆盖** base。所以 base 里放通用结构，config 里放训练相关量。

### 1.6 FSM 注册契约

```cpp
// fsm.hpp
std::string type = factory->GetType();          // 注册时的键
factories_[type] = factory;

// rl_sim_mujoco.cpp:88
if (FSMManager::GetInstance().IsTypeSupported(this->robot_name))
    auto fsm_ptr = FSMManager::GetInstance().CreateFSM(this->robot_name, this);
else
    std::cout << LOGGER::ERROR << "[FSM] No FSM registered for robot: " << this->robot_name;
    // ← 只打印，然后继续初始化！机器人没有状态机，所有电机收不到指令，瘫在地上
```

**`GetType()` 的返回值必须逐字符等于命令行的第 1 个参数。**

注册方式（头文件，无需改 CMakeLists）：

1. 复制 `src/rl_sar/fsm_robot/fsm_go2.hpp` → `fsm_<robot>.hpp`
2. 全局替换：namespace、工厂类名、include guard、`GetType()` 返回的字符串
3. 在 `src/rl_sar/fsm_robot/fsm_all.hpp` 里加 `#include "fsm_<robot>.hpp"`

`fsm_all.hpp` 被 [`src/rl_sar/include/rl_sim_mujoco.hpp`](../src/rl_sar/include/rl_sim_mujoco.hpp) 包含，工厂在**静态初始化期**注册。纯头文件**不需要**加进 CMakeLists。

---

## 2. 七步移植流程

> 每步做完**必须验证再进入下一步**。前一步的错误会伪装成后一步的症状。

### Step 0 · 复制描述目录

```bash
cp -r src/rl_sar_zoo/go2_description src/rl_sar_zoo/<robot>_description
```

然后替换 `urdf/`、`mjcf/`、`meshes/` 为自己的模型。**保留目录名 `<robot>_description`**（契约 1.1）。

<details>
<summary>排除：只是复制来抄格式的残留文件</summary>

`package.ros1.xml`、`package.ros2.xml`、`CMakeLists.txt`（`project(go2_description ...)`）、`launch/`、`config/`、`xacro/` 里的 go2 字样**不影响 MuJoCo**，只影响 Gazebo/RViz。可留到最后统一清理（见附录 A）。

</details>

---

### Step 1 · 出生高度

**目的**：机器人初始时脚底正好贴地，不穿地、不悬空。

**改**：`mjcf/<robot>.xml` 里 torso 的 `pos` 的 z 分量。

```xml
<body name="torso" pos="0 0 0.50">
```

**怎么算**：`z = 髋关节到脚底的总偏移`。

对四足，把从髋到足端碰撞体底部的 z 偏移累加：

```
0.23        (hip → thigh)
+ 0.2315648 (thigh → toe)
+ 0.012     (toe → 足端碰撞体中心)
+ 0.027     (足端碰撞体半径)
= 0.5006  → 取 0.50
```

**验证**：

```bash
cd <repo>
python3 -c "
import mujoco
m = mujoco.MjModel.from_xml_path('src/rl_sar_zoo/<robot>_description/mjcf/<scene>.xml')
d = mujoco.MjData(m); mujoco.mj_forward(m, d)
lo = min(d.geom_xpos[g][2] - m.geom_size[g][0] for g in range(m.ngeom) if m.geom_type[g] == mujoco.mjtGeom.mjGEOM_CYLINDER)
print('足端最低点 z =', round(lo, 4), ' (应接近 0，略大或略小于 0 都可接受)')
"
```

---

### Step 2 · 传感器重排（★核心）

**目的**：满足契约 1.2。

**改**：`mjcf/<robot>.xml` 的 `<sensor>` 块，重排成严格的四段式：

```xml
<sensor>
  <!-- 第一段：N 个关节位置，顺序 = <actuator> 顺序 -->
  <jointpos name="fl_hip_roll_joint_pos"  joint="fl_hip_roll_joint"  />
  <jointpos name="fl_hip_pitch_joint_pos" joint="fl_hip_pitch_joint" />
  <jointpos name="fl_knee_joint_pos"      joint="fl_knee_joint"      />
  <!-- … 共 N 个 … -->

  <!-- 第二段：N 个关节速度，顺序同上 -->
  <jointvel name="fl_hip_roll_joint_vel"  joint="fl_hip_roll_joint"  />
  <!-- … 共 N 个 … -->

  <!-- 第三段：N 个关节力矩，顺序同上 -->
  <jointactuatorfrc name="fl_hip_roll_joint_torque" joint="fl_hip_roll_joint" />
  <!-- … 共 N 个 … -->

  <!-- 第四段：IMU -->
  <framequat name="body_quat" objtype="site" objname="imu" />
  <gyro      name="body_gyro" site="imu" />
  <accelerometer name="body_acc" site="imu" />   <!-- 可选，代码不读 -->
  <framepos name="body_pos" objtype="site" objname="imu" />  <!-- 可选 -->
</sensor>
```

**关键点**：

1. **三段之间不能交错**。原始模型常见的是"每个关节一组 pos/vel/torque 交错"，必须拆开。
2. `framequat` / `gyro` 前面**必须**先有 `3N` 个标量。
3. **第三段不能漏**。原模型常常只有 pos/vel 没有 torque，这会导致 `nsensordata` 少 N 个 → 契约 1.2 说的越界读。
4. XML 里元素名是 `<jointactuatorfrc>`，但 C 枚举是 `mjSENS_JOINTACTFRC`——**名字不一致是 MuJoCo 的历史遗留**，写脚本时别搞混。

**验证**：用 §3 的**脚本一**。这是整个移植过程最重要的一次验证。

**期望**（N=12）：`nsensordata = 49`，且 `sensor_adr` 36→framequat、40→gyro。

---

### Step 3 · 物理参数

**目的**：让仿真稳定、接触可信。这一步不涉及契约，纯物理。

**改**：`mjcf/<robot>.xml`

**3a. 接触求解器**（放在 `<mujoco>` 下）

```xml
<option cone="elliptic" impratio="100" />
```

`cone="elliptic"` 比默认的 `pyramidal` 更接近真实摩擦锥；`impratio` 提高法向/切向阻抗比，减少打滑。

**3b. 关节默认阻尼与电枢**

```xml
<default>
  <joint damping="0.1" armature="0.01" frictionloss="0.2" />
</default>
```

`armature` 模拟转子惯量（对 sim2real 很重要），`damping` 和 `frictionloss` 模拟减速器摩擦。

> ⚠️ **不要照抄 go2 的 `<geom condim="1" .../>` 默认值。**
> go2 用 `childclass="go2"` 的类继承体系，足端子类会覆盖成 `condim="6"`。
> 如果你的模型是显式写 geom、没有类继承，全局 `condim="1"` 会让**脚变成无摩擦**，机器人直接滑走。

**3c. 足端摩擦与优先级**

```xml
<geom name="fl_foot_collision" type="cylinder" size="0.027 0.012"
      friction="0.6 0.005 0.0001" priority="1" />
```

`priority="1"` 让足端摩擦覆盖地面的摩擦设置（MuJoCo 取 priority 高的一方的 friction）。

**验证**：

```bash
cd <repo>
python3 -c "
import mujoco
m = mujoco.MjModel.from_xml_path('src/rl_sar_zoo/<robot>_description/mjcf/<scene>.xml')
print('cone   =', m.opt.cone, '(1 = elliptic)')
print('impratio =', m.opt.impratio)
print('armature =', m.dof_armature[:12])
print('damping  =', m.dof_damping[:12])
feet = [g for g in range(m.ngeom) if 'foot' in (m.geom(g).name or '')]
print('foot geom condim:', [int(m.geom_condim[g]) for g in feet], '(应为 3 或 6，绝不能是 1)')
"
```

---

### Step 4 · 分离 `scene.xml`

**目的**：把"机器人"和"场景"分开，便于换地面/光照做不同实验。

**为什么必须做**：原 `go2.xml` 自带 `<geom name="floor">` 和 `<light>`。如果直接照搬 go2 的 `scene.xml`，会出现**两层重叠地面**。

**改**：

1. 用 `go2_description/mjcf/scene.xml` 为模板，新建 `mjcf/<scene>.xml`：

```xml
<mujoco model="<robot> scene">
  <include file="<robot>.xml" />
  <statistic center="0 0 0.1" extent="0.8" />
  <visual>
    <headlight diffuse="0.6 0.6 0.6" ambient="0.3 0.3 0.3" specular="0 0 0" />
    <rgba haze="0.15 0.25 0.35 1" />
    <global azimuth="-130" elevation="-20" />
  </visual>
  <asset>
    <texture type="skybox" builtin="gradient" rgb1="0.3 0.5 0.7" rgb2="0 0 0" width="512" height="3072" />
    <texture type="2d" name="groundplane" builtin="checker" mark="edge"
             rgb1="0.2 0.3 0.4" rgb2="0.1 0.2 0.3" markrgb="0.8 0.8 0.8" width="300" height="300" />
    <material name="groundplane" texture="groundplane" texuniform="true" texrepeat="5 5" reflectance="0.2" />
  </asset>
  <worldbody>
    <light pos="0 0 1.5" dir="0 0 -1" directional="true" />
    <geom name="floor" size="0 0 0.05" type="plane" material="groundplane" />
  </worldbody>
</mujoco>
```

2. **从 `<robot>.xml` 中删掉** `<light name="sun" .../>` 和 `<geom name="floor" .../>`。

3. **地面必须用无限平面** `size="0 0 0.05"`（前两个分量会被忽略）。
   **不要用 `size="3 3 0.1"`** ——那是 6×6 m 的有界平面，机器人走几步就掉下去。

关于 `<include>`：跨 include 边界的多个 `<asset>` / `<worldbody>` 段会**合并**，这是 MuJoCo 的既定语义，无需额外处理。

**验证**：用 §3 的**脚本二**。必须**恰好 1 个** plane geom 和 **1 个** light。

---

### Step 5 · `policy/<robot>/base.yaml`

**目的**：填入通用结构参数。**这是最优先的一步**——因为契约 1.5 里说了，文件缺失会导致 `num_of_dofs=0`、`dt=0` 的静默瘫痪。

**为什么必须先于 FSM 做**：`ReadYaml(robot_name, "base.yaml")` 在 `RL_Sim` 构造早期就调用了，缺文件不报错，后面所有步骤的表现都会被这个静默失败污染。

**改**：新建 `policy/<robot>/base.yaml`

```yaml
<robot>:
  dt: 0.005            # 控制周期，200 Hz
  decimation: 4        # 策略周期 = dt * decimation = 50 Hz
  num_of_dofs: 12
  wheel_indices: []    # 有轮机器人填轮子下标，四足填 []

  # 以下向量的顺序 = mjcf 的 <actuator> / <sensor> 顺序（见契约 1.4）
  fixed_kp: [...]      # N 项，GetUp/GetDown 插值用
  fixed_kd: [...]
  torque_limits: [...] # 与 urdf 的 effort= 和 mjcf 的 ctrlrange= 一致
  default_dof_pos: [...]  # 站立姿态，顺序同上

  joint_names: [...]              # MuJoCo 不读，仅供可读性 / Gazebo
  joint_controller_names: [...]   # 仅 Gazebo / CSV_LOGGER 用
  joint_mapping: [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11]   # 恒等
```

#### 哪些字段是 MuJoCo 路径真正读的

| 字段 | 谁在用 | 必需 |
|---|---|---|
| `dt` / `decimation` | 控制循环周期 | ✅ |
| `num_of_dofs` | `InitJointNum` | ✅ |
| `joint_mapping` | `GetState` / `SetCommand` | ✅ |
| `default_dof_pos` | GetUp 插值目标 + 观测基准 | ✅ |
| `fixed_kp` / `fixed_kd` | `Interpolate()`（GetUp/GetDown，默认 `use_fixed_gains=true`） | ✅ |
| `torque_limits` | `ComputeOutput` 的 clamp | ✅ |
| `rl_kp` / `rl_kd` | `RLControl()`（RL 模式） | 由策略 config 提供 |
| `joint_names` | **只在 `rl_sim.cpp`（Gazebo）用** | ❌ |
| `joint_controller_names` | 只在 `CSV_LOGGER` 分支用 | ❌ |

#### `default_dof_pos` 怎么定

这是"站立姿态"，是纯物理量，**没有通用公式**。确定方法：

1. **看关节限位**。膝盖限位通常是全负（如 `[-2.75, -0.55]`），说明膝朝后弯，站立值取中间偏折的位置（如 `-1.5`）。
2. **看轴向**。`axis="(1,0,0)"` 的髋外展取 0；`axis="(0,1,0)"` 的大腿取正值（0.8 左右）。
3. **如果新机器人的关节轴向和拓扑与 go2 一致**（髋 roll 轴 `(1,0,0)`、大腿/膝轴 `(0,1,0)`、hip→thigh 沿 ±y、thigh→calf 沿 −z），那么 go2 的 `[0.0, 0.8, -1.5]` 可以直接作为起点。
4. **最终靠 Step 6 目视确认**：四条腿应大致对称、膝朝后、脚掌贴地、身体离地约等于 Step 1 的出生高度。

**验证**：用 §3 的**脚本三**（校验 base.yaml 顺序 vs mjcf 顺序，以及长度一致性）。

---

### Step 6 · `fsm_<robot>.hpp` 并注册

**目的**：满足契约 1.6。

**改**：

1. **复制** `src/rl_sar/fsm_robot/fsm_go2.hpp` → `src/rl_sar/fsm_robot/fsm_<robot>.hpp`（原地保留 go2 那份）。

2. **全局替换**（顺序按表格从上往下，避免误伤）：

   | 查找 | 替换为 | 位置 |
   |---|---|---|
   | `Go2FSMFactory` | `<Robot>FSMFactory` | 类名 / 构造函数 / 宏参数 |
   | `go2_fsm` | `<robot>_fsm` | namespace 开闭 + `go2_fsm::` 限定 |
   | `GO2_FSM_HPP` | `<ROBOT>_FSM_HPP` | include guard 两处 |
   | `"go2"` | `"<robot>"` | `GetType()` 返回值，**必须带引号替换** |

3. **语义修改 a**：`rl.config_name = "himloco";` → 改成你自己的策略目录名（如 `"loco"`）。
   这个字符串要和 Step 7 建的目录名**一字不差**。

4. **语义修改 b**：`RLFSMStateGetUp::pre_running_pos` 改成**恰好 N 个值**，顺序同 base.yaml。

   ```cpp
   std::vector<float> pre_running_pos = {
       0.00, 1.36, -2.65,   // 腿 1
       0.00, 1.36, -2.65,   // 腿 2
       0.00, 1.36, -2.65,   // 腿 3
       0.00, 1.36, -2.65    // 腿 4
   };
   ```

   含义：起身第一步的**折叠姿态**（先把腿收起来撑住，再展开到 `default_dof_pos`）。
   go2 原版有 16 个值（多出 4 个 `0.00`），**不会崩**——`Interpolate` 循环的是 `i < num_of_dofs`，多出来的从不访问。但为可读性建议清掉。
   数值仍是纯物理量，先抄 go2 再看效果。

5. **注册**：在 `src/rl_sar/fsm_robot/fsm_all.hpp` 里加一行 `#include "fsm_<robot>.hpp"`（位置随意，`factories_` 是按名字查的 map）。

6. **重新编译**（头文件变了必须重编）：

   ```bash
   cd <repo> && cmake --build cmake_build -j$(nproc)
   ```

**验证**：

```bash
cd <repo>
LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libstdc++.so.6 \
  ./cmake_build/bin/rl_sim_mujoco <robot> <scene> 2>&1 | tee /tmp/port.log
```

期望前几行出现：

```
[FSMManager] Registered type: <robot>
[FSMManager] FSM created for type: <robot>
```

**以及不再出现**：

```
[FSM] No FSM registered for robot: <robot>
```

**按键验收**：

| 按键 | 期望 |
|---|---|
| 启动瞬间 | 机器人**瘫倒在地** —— 正确。初始态 Passive 里 `kp=0, kd=8`，就是卸力状态 |
| `0` | GetUp：先收到 `pre_running_pos` 折叠姿态，再插值到 `default_dof_pos`，**四条腿撑住、身体抬起** |
| `9` | GetDown，插值回初始姿态 |
| `1` | 进 RL 模式。此刻 `policy/<robot>/<cfg>/` 还不存在，**预期报错并弹回 Passive** —— 这不是失败 |

**这一屏同时验收 Step 1 / 5 / 6**：站起来说明 `default_dof_pos` 和出生高度是对的；站姿歪斜/劈叉/发抖说明要回头调 `default_dof_pos` 和 `pre_running_pos`。

---

### Step 7 · 策略配置 + 模型

**目的**：让 `1` 键真正跑起策略。

#### 7a. `policy/<robot>/<cfg>/config.yaml`

选一份官方模板抄结构。两份参考的差异：

| | `go2/robot_lab` | `go2/himloco` |
|---|---|---|
| 观测历史 | `[]`（输入 45 维） | `[0..5]`（输入 270 维） |
| `joint_mapping` | 恒等 | 非恒等 |
| 契约复杂度 | **低** | 高 |

**建议先用 `robot_lab` 那份**，契约简单，跑通再上复杂配置。

**顶层 key 必须是 `<robot>/<cfg>`**（契约 1.5）。

字段分三类：

```yaml
<robot>/<cfg>:
  # ---- 第一类：机器人固有量，由 Step 1-6 定，与训练无关 ----
  num_of_dofs: 12
  torque_limits: [30.0, ...]        # 与 urdf effort / mjcf ctrlrange 一致
  default_dof_pos: [ 0.0, 0.8, -1.5, ...]

  # ---- 第二类：★必须与训练配置一字不差，否则 sim2sim 必翻车 ----
  model_name: "policy.pt"
  num_observations: 45
  observations: ["ang_vel", "gravity_vec", "commands", "dof_pos", "dof_vel", "actions"]
  observations_history: []
  observations_history_priority: "time"
  lin_vel_scale: 2.0
  ang_vel_scale: 0.25
  dof_pos_scale: 1.0
  dof_vel_scale: 0.05
  commands_scale: [1.0, 1.0, 1.0]
  action_scale: [0.125, 0.25, 0.25, ...]
  rl_kp: [20.0, ...]
  rl_kd: [0.5, ...]
  joint_mapping: [0, 1, ..., 11]    # ← 见下

  # ---- 第三类：可自由调 ----
  default_commands: [0.0, 0.0, 0.0]
  clip_obs: 100.0
  clip_actions_lower: [-100.0, ...]
  clip_actions_upper: [100.0, ...]
  fixed_kp: [80.0, ...]             # 覆写 base.yaml，只影响起身
  fixed_kd: [3.0, ...]
  wheel_indices: []
```

**`num_observations` 必须等于 `observations` 各项维度之和**：

```
ang_vel      3
gravity_vec  3
commands     3
dof_pos      N
dof_vel      N
actions      N
           ────
         9 + 3N      → N=12 时 = 45
```

若 `observations_history` 非空，则策略**输入**维度 = `num_observations × len(observations_history)`。

**`joint_mapping` 怎么算**：

> `joint_mapping[i]` = 训练顺序里第 `i` 个关节，在 mjcf 里的下标。

**这一步必须从训练侧的 `joint_names` 反推，不能照抄 URDF 文档顺序** —— 完整方法与验收见 1.4.1。

以 bpx 为例（mjcf / base.yaml 顺序）：

```
 0 fl_hip_roll   1 fl_hip_pitch   2 fl_knee
 3 fr_hip_roll   4 fr_hip_pitch   5 fr_knee
 6 hl_hip_roll   7 hl_hip_pitch   8 hl_knee
 9 hr_hip_roll  10 hr_hip_pitch  11 hr_knee
```

- **bpx 实测（按关节类型分组，腿序 fl→fr→hl→hr）** → `[0,3,6,9, 1,4,7,10, 2,5,8,11]`
  - 配套 `default_dof_pos`：`[0,0,0,0, 0.8,0.8,0.8,0.8, -1.5,-1.5,-1.5,-1.5]`
  - **两者必须同时改**，只改一个会在站立时就侧翻
- 训练顺序与 mjcf 相同（**少见**，别默认）→ `[0,1,2,3,4,5,6,7,8,9,10,11]`
- go2 的 himloco（IsaacLab 自带 USD，顺序 FL/FR/RL/RR，每条腿 roll-pitch-knee）
  → `[3,4,5, 0,1,2, 9,10,11, 6,7,8]`

[README.md](../README.md) 里的官方口径可以直接用：

> The order of joints in robot_lab cfg file `joint_names` is the same as that defined in `xxx/robot_lab/config.yaml` in this project.

即：**训练框架 cfg 里的 `joint_names` 列表顺序 = 本 config.yaml 的顺序**，据此换算成 `joint_mapping`。

**从训练资产 cfg 抄三个增益值**（与关节顺序无关，但同样属于"必须与训练一致"）：

| config.yaml | 训练侧来源 | bpx/himloco 实测 |
|---|---|---|
| `rl_kp` | `DCMotorCfg.stiffness` | `45.0` |
| `rl_kd` | `DCMotorCfg.damping` | `1.2` |
| `torque_limits` | `effort_limit`（与 mjcf `ctrlrange` 一致） | `30.0` |

> ⚠️ `rl_kd` 很容易被想当然写成 0.5 之类的小值。它就是训练的 `damping`，
> 写小了仿真会抖、会晃，并不是"更软更安全"。

#### 7b. `.pt` 模型

- **格式必须是 TorchScript**（`torch.jit.script` 导出），不是 `state_dict`。
  加载走 `InferenceRuntime::TorchModel`（`torch::jit::script::Module`），也支持 ONNX 自动识别。
- 输入维必须匹配：`num_observations × len(observations_history)`。
- **不要复制别的机器人的 `.pt` 充数**。输入维度可能恰好相同，但动力学不同，必然摔倒，还会掩盖真实问题。

#### 7c. 验证

**不写模型也能验证一半**——故意让模型缺失，看报错路径：

```bash
cd <repo>
LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libstdc++.so.6 \
  ./cmake_build/bin/rl_sim_mujoco <robot> <scene> 2>&1 | tee /tmp/port.log
# 按 0 起身，再按 1
```

| 报错里的路径 | 结论 |
|---|---|
| `.../policy/<robot>/<cfg>/policy.pt` | ✅ config 读到了，只缺模型文件 |
| `.../policy/<robot>/<cfg>/`（斜杠结尾） | ❌ **顶层 key 写错了**，回去改成 `<robot>/<cfg>`，并确认它在文件里顶格、无缩进 |
| 无 `InitRL() failed`，直接弹回 Passive 且无输出 | ❌ config.yaml 文件不存在或路径不对 |

放入 `.pt` 后再按 `1`，应看到：

```
RL Controller [<cfg>] x:0 y:0 yaw:0
```

按 `W/A/S/D` 给速度指令，机器人应迈步。

---

## 3. 验证脚本合集

复制即用。全部只读，不修改任何文件。

### 脚本一 · 传感器布局校验（最重要）

```bash
cd <repo>
python3 - <<'PY'
import mujoco
ROBOT, SCENE, N = "bpx", "scene", 12          # ← 改这里
m = mujoco.MjModel.from_xml_path(f'src/rl_sar_zoo/{ROBOT}_description/mjcf/{SCENE}.xml')
S = mujoco.mjtSensor

def check(off, typ, want):
    for i in range(m.nsensor):
        if m.sensor_adr[i] == off:
            if m.sensor_type[i] == typ:
                print(f'  adr {off:3d}  {m.sensor(i).name:<28s} OK')
                return True
            print(f'  adr {off:3d}  {m.sensor(i).name:<28s} 类型错误 (期望 {want})')
            return False
    print(f'  adr {off:3d}  <缺失>  (期望 {want})')
    return False

ok = True
for i in range(N):
    ok &= check(i,         S.mjSENS_JOINTPOS,  'jointpos')
    ok &= check(i + N,     S.mjSENS_JOINTVEL,  'jointvel')
    ok &= check(i + 2 * N, S.mjSENS_JOINTACTFRC, 'jointactuatorfrc')
ok &= check(3 * N,     S.mjSENS_FRAMEQUAT, 'framequat')
ok &= check(3 * N + 4, S.mjSENS_GYRO,      'gyro')

print(f'\nnsensordata = {m.nsensordata}  (最少需要 {3*N+7})')
print('=> 布局正确' if ok and m.nsensordata >= 3 * N + 7 else '=> 布局错误，禁止继续')
PY
```

### 脚本二 · 场景唯一性校验

```bash
cd <repo>
python3 - <<'PY'
import mujoco
ROBOT, SCENE = "bpx", "scene"                 # ← 改这里
m = mujoco.MjModel.from_xml_path(f'src/rl_sar_zoo/{ROBOT}_description/mjcf/{SCENE}.xml')
G = mujoco.mjtGeom
planes = [m.geom(g).name for g in range(m.ngeom) if m.geom_type[g] == G.mjGEOM_PLANE]
print('plane geom  =', planes, '  <- 必须恰好 1 个')
print('nlight      =', m.nlight, '  <- 期望 1')
print('nq          =', m.nq, '  <- 期望 7 + N')
print('nu          =', m.nu, '  <- 期望 N')
print('nsensordata =', m.nsensordata)
PY
```

### 脚本三 · base.yaml 与 mjcf 顺序对齐校验

```bash
cd <repo>
python3 - <<'PY'
import mujoco, yaml
ROBOT, SCENE = "bpx", "scene"                 # ← 改这里
m = mujoco.MjModel.from_xml_path(f'src/rl_sar_zoo/{ROBOT}_description/mjcf/{SCENE}.xml')
c = yaml.safe_load(open(f'policy/{ROBOT}/base.yaml'))[ROBOT]
N = c['num_of_dofs']

# 1) 所有 12 维向量长度一致
bad = [k for k, v in c.items() if isinstance(v, list) and len(v) != N and k != 'joint_mapping']
print('长度不为', N, '的字段:', bad or '无')

# 2) joint_names 顺序 == mjcf 顺序
names, ok = c['joint_names'], True
for i, n in enumerate(names):
    sens = m.sensor(i).name
    act = m.joint(m.actuator_trnid[i, 0]).name
    good = (sens == n + '_pos') and (act == n)
    ok &= good
    print(f'[{i:2d}] base={n:<24s} sensor={sens:<28s} actuator={act:<24s} {"OK" if good else "MISMATCH"}')
print('=> 三者完全对齐' if ok else '=> 顺序不一致，必须重排 base.yaml')
PY
```

### 脚本四 · 出生高度 / 足底最低点

见 Step 1。

---

## 4. 排错速查表

| 症状 | 最可能的原因 | 检查 |
|---|---|---|
| `[FSM] No FSM registered for robot: X` | Step 6 未做，或 `GetType()` 字符串与命令行参数不一致 | `fsm_all.hpp` 里加了 include 吗？重新编译了吗？ |
| `[FSMManager] Error: Unsupported type: X` | 同上，但注册表里根本没有该键 | 同上 |
| 机器人开机即瘫软，且**无任何报错** | `policy/<robot>/base.yaml` 缺失或顶层 key 写错 → `num_of_dofs=0`、`dt=0` | 看启动日志有无 `ReadYaml` 的错误行；用脚本三 |
| 机器人起步即抽搐/抖动/乱窜 | 传感器布局不对（**越界读**） | 脚本一，必须是"布局正确" |
| 按 `1` 报 `Failed to load model from: .../<cfg>/`（斜杠结尾） | config.yaml 顶层 key 写错（应为 `<robot>/<cfg>`） | 见 Step 7c |
| 按 `1` 报 `Failed to load model from: .../x.pt` | 模型文件不存在或不是 TorchScript | 文件在吗？`torch.jit.load()` 能加载吗？ |
| 站姿歪斜 / 劈叉 | `default_dof_pos` 不对 | Step 5 的确定方法；目视调 |
| 起身时某条腿撞限位 / 抽搐 | `pre_running_pos` 不对 | Step 6 语义修改 b |
| 机器人滑走 | 足端 `condim="1"`（无摩擦）或 `priority` 没设 | Step 3c |
| 走两步掉出地面 | 地面用了有界平面 | Step 4，改 `size="0 0 0.05"` |
| 启动即穿地或悬空 | 出生高度不对 | Step 1 |
| 策略跑起来但原地不动 / 动作幅度极小 | `action_scale` 或 `rl_kp` 与训练不一致 | Step 7a 第二类字段 |
| 策略跑起来但完全乱动 | `joint_mapping` 与训练不一致 | 1.4.1 的换算规则 |
| 按 `1` 后**侧翻 / 肚皮朝上** | `joint_mapping` 与 `default_dof_pos` 不配对（多半是同一条腿内 roll/pitch/knee 的次序错） | 1.4.1 的"两个测试"；两者必须同时改 |
| 能站住，一给速度指令就横走 / 打转 | 腿与腿之间的次序错 | 1.4.1，换一种腿排列 |
| 站立/行走时发抖、晃动不止 | `rl_kd` 比训练的 `damping` 小 | Step 7a 的增益表 |

---

## 5. 已知坑（本次踩过的）

### 5.1 环境类

**conda 的 libstdc++ 冲突**

症状：运行时 `libtinfo.so.6: no version information available`，或二进制启动即崩。

```bash
LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libstdc++.so.6 ./cmake_build/bin/rl_sim_mujoco <robot> <scene>
```

**`| head -N` 会吞掉输出**

管道下 `std::cout` 是**全缓冲**的。`| head -30` 会让后续所有日志卡在缓冲区里看不到（只有走 `std::endl` 的行会 flush）。调试时用：

```bash
... 2>&1 | tee /tmp/port.log
```

### 5.2 契约类

| 坑 | 后果 |
|---|---|
| 传感器按"每个关节一组"交错排列 | 越界读，机器人乱动 |
| 漏掉 `<jointactuatorfrc>` 段 | `nsensordata` 少 N 个 → 越界读 |
| 按**传感器元素序号**而非 **`sensor_adr`** 数偏移 | 从 IMU 开始全错 |
| 执行器写成 `<position>` 而非 `<motor>` | PD 被算了两次，完全失控 |
| config.yaml 顶层 key 写成 `<cfg>` 而非 `<robot>/<cfg>` | `model_name` 变空，路径退化 |
| 照抄 go2 的全局 `<geom condim="1">` | 脚无摩擦，机器人滑走 |
| 直接抄 go2 的 `scene.xml` 而不删原模型的 floor/light | 两层重叠地面 |
| 地面用 `size="3 3 0.1"` | 6×6 m 有界，走两步掉下去 |
| 复制描述目录时改了 `<robot>_description` 的名字 | 模型找不到 |
| 复制别的机器人的 policy 目录后没改顶层 key | 见 5.2 第三行 |
| 复制别的机器人的 `.pt` | 必然摔倒，且掩盖真实问题 |
| 拿 URDF 文档顺序当训练关节顺序（想当然填恒等映射） | 站立就侧翻 / 肚皮朝上 |
| 只改 `joint_mapping` 不改 `default_dof_pos` | 同上——两者同源，必须一起改 |
| `rl_kd` 抄成 0.5 这类小值（训练的 `damping` 是 1.2） | 仿真发抖、晃动 |

### 5.3 设计类

**`base.yaml` vs 策略 `config.yaml` 的分层**

正确做法（官方 go2 就是这么分的）：

- `base.yaml`：`dt`、`decimation`、`num_of_dofs`，以及**按 mjcf 顺序**排列的 `fixed_kp/kd`、`torque_limits`、`default_dof_pos`，`joint_mapping` 恒等。
- 策略 `config.yaml`：所有**训练相关量**（`rl_kp/kd`、`action_scale`、各 scale、`observations*`），以及承载"训练顺序 → mjcf 顺序"的 `joint_mapping`。

这样换一份策略只需要改 config.yaml，`base.yaml` 不动。

**`rl_kp/rl_kd` vs `fixed_kp/fixed_kd` 用在哪**

- `RLControl()`（RL 模式，按 `1`）→ 用 `rl_kp` / `rl_kd`
- `Interpolate()`（GetUp/GetDown，按 `0` / `9`）→ 用 `fixed_kp` / `fixed_kd`（默认 `use_fixed_gains=true`）

调起身刚度改 `fixed_*`，调行走刚度改 `rl_*`。

**`ang_vel_axis` 在 MuJoCo 下固定为 `"body"`**

在 `RL_Sim` 构造函数里硬编码，不受 YAML 控制（Gazebo ROS1 才是 `"world"`）。所以训练时角速度观测必须用机体系。

**控制频率**

`dt × decimation = 策略周期`。`dt=0.005, decimation=4` → 200 Hz 控制、50 Hz 策略。改这个必须同步改训练配置的训练频率。

---

## 附录 A · 非 MuJoCo 路径还需要改什么

本次未做。若将来要接 Gazebo（`rl_sim`）或真机，还需处理：

| 文件 | 需要改什么 |
|---|---|
| `<robot>_description/CMakeLists.txt` | `project(go2_description ...)` → `<robot>_description` |
| `<robot>_description/package.ros1.xml` | `<description>` 里的 go2 字样 |
| `<robot>_description/package.ros2.xml` | 同上 |
| `<robot>_description/launch/gazebo.launch` | `$(find go2_description)`、urdf 文件名、`-model` 参数 |
| `<robot>_description/launch/go2_rviz.launch` | 文件名、`$(find ...)` |
| `<robot>_description/config/robot_control.yaml` | 顶层 key `go2_gazebo:` 和关节名 |
| `<robot>_description/xacro/gazebo.xacro` | 传动/插件配置 |
| `<robot>_description/xacro/robot.xacro` | 引用路径 |
| `src/rl_sar/src/rl_real_<robot>.hpp/.cpp` | 真机 SDK 对接（新建） |

另外注意：`base.yaml` 里的 `joint_names` / `joint_controller_names` **只有 Gazebo 路径才读**，接 Gazebo 时必须填对。

---

## 附录 B · bpx 本次的实际落地值

可直接对照，作为下一次的参考基准。

| 项 | 值 |
|---|---|
| 机器人名 | `bpx` |
| 场景名 | `scene` |
| 自由度数 N | 12 |
| 描述目录 | `src/rl_sar_zoo/bpx_description/` |
| 关节顺序（mjcf / `base.yaml`） | `fl_*`, `fr_*`, `hl_*`, `hr_*`，每腿 `hip_roll` / `hip_pitch` / `knee` |
| 关节轴 | 髋 roll `(1,0,0)`；大腿、膝 `(0,1,0)` |
| 膝限位 | `[-2.7531, -0.5539]` |
| 出生高度 | `pos="0 0 0.50"`（计算值 0.5006） |
| `nsensordata` | 49（= 3×12 + 7 + 3 acc + 3 framepos） |
| `nq` / `nu` | 19 / 12 |
| `torque_limits` | 30.0（urdf `effort="30"`，mjcf `ctrlrange="-30 30"`） |
| `default_dof_pos` | `[0.0, 0.8, -1.5]` × 4 |
| `fixed_kp` / `fixed_kd` | 80.0 / 3.0 |
| `dt` / `decimation` | 0.005 / 4 |
| `joint_mapping` | `[0..11]` 恒等 |
| 物理参数 | `cone="elliptic" impratio="100"`；关节 `damping=0.1 armature=0.01 frictionloss=0.2`；足端 `friction="0.6 0.005 0.0001" priority="1"` |
| FSM 文件 | `src/rl_sar/fsm_robot/fsm_bpx.hpp` |
| FSM 类型串 | `"bpx"` |

以上是 **`base.yaml`**（= mjcf 顺序）的落地值。**策略 `config.yaml` 的那份是另一套**，
它由训练侧决定，实测（himloco_lab 训出的 `bpx_rough/policy.pt`）：

| strategy `config.yaml` | 值 |
|---|---|
| 训练关节顺序 | 按关节类型分组，腿序 fl→fr→hl→hr |
| `joint_mapping` | `[0, 3, 6, 9,  1, 4, 7, 10,  2, 5, 8, 11]` |
| `default_dof_pos` | `[0,0,0,0, 0.8,0.8,0.8,0.8, -1.5,-1.5,-1.5,-1.5]` |
| `rl_kp` / `rl_kd` | `45.0` / `1.2`（训练 `stiffness` / `damping`） |
| `torque_limits` | `30.0`（训练 `effort_limit`） |
| `action_scale` / `commands_scale` | `0.25` × 12 / `[1,1,1]` |
| `observations_history` | `[0..5]`（训练 `history_length=5` → 6 帧 × 45 = 270 维输入） |

> ⚠️ 这里的腿序（fl→fr→hl→hr）是**按 URDF link 声明顺序 + 层序遍历推出来的，尚未用
> `robot.joint_names` 直接确认**。它只能解释"站得住"，不能解释"走得直"——若实测
> 给正向指令后横走/打转，改用 `[3, 0, 9, 6, 4, 1, 10, 7, 5, 2, 11, 8]`（腿序 fr→fl→hr→hl）。

---

## 一句话总结

> 移植的全部难点在于 **`rl_sim_mujoco` 用下标而非名字对接模型**。
> 把 `<sensor>` 排成 `N×pos | N×vel | N×torque | quat | gyro`，
> 让 `<actuator>` 顺序与之相同，
> 用 `sensor_adr`（不是元素序号）校验，
> 把 YAML 的两个顶层 key 写对，
> 剩下的事就都是可查表解决的。
>
> 唯一的例外是**训练侧的关节顺序**——它不在你的控制范围内，
> 只能从 `robot.joint_names` 读出来，
> 并按同一次序推出 `joint_mapping` 和 `default_dof_pos`（见 1.4.1）。
