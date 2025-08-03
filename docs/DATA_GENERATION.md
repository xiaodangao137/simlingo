# SimLingo 数据生成详细文档 (Data Generation Detailed Documentation)

本文档详细介绍了 SimLingo 项目中的数据生成管道，包括驾驶数据收集、语言标签生成、梦想数据创建以及数据增强等各个组件。

This document provides a comprehensive guide to the data generation pipeline in the SimLingo project, covering driving data collection, language label generation, dreamer data creation, and data augmentation components.

## 目录 (Table of Contents)

1. [概览 (Overview)](#概览-overview)
2. [驾驶数据生成 (Driving Data Generation)](#驾驶数据生成-driving-data-generation)
3. [语言数据生成 (Language Data Generation)](#语言数据生成-language-data-generation)
4. [梦想数据生成 (Dreamer Data Generation)](#梦想数据生成-dreamer-data-generation)
5. [数据分桶 (Data Buckets)](#数据分桶-data-buckets)
6. [数据增强 (Data Augmentation)](#数据增强-data-augmentation)
7. [故障排除 (Troubleshooting)](#故障排除-troubleshooting)
8. [最佳实践 (Best Practices)](#最佳实践-best-practices)

## 概览 (Overview)

SimLingo 的数据生成管道包含四个主要阶段：

```
1. 驾驶数据收集      →    2. 语言标签生成      →    3. 梦想数据创建      →    4. 数据增强
   [CARLA + PDM-Lite]    [VQA + Commentary]    [Action Dreaming]       [ChatGPT]
```

### 数据流向 (Data Flow)

1. **驾驶数据收集**: 使用 PDM-Lite 专家在 CARLA 模拟器中收集驾驶轨迹
2. **语言标签生成**: 基于收集的驾驶数据生成 VQA 和解说数据
3. **梦想数据创建**: 生成多种可能的未来轨迹用于语言-动作对齐
4. **数据增强**: 使用 ChatGPT 扩展语言模板以增加多样性

### 依赖要求 (Dependencies)

- CARLA 0.9.15
- Python 3.8+
- SLURM cluster (推荐用于大规模数据收集)
- OpenAI API key (用于数据增强)
- 足够的存储空间 (建议 > 1TB)

### 配置示例 (Configuration Examples)

参考 [docs/EXAMPLE_CONFIG.md](EXAMPLE_CONFIG.md) 获取详细的配置示例和模板。
See [docs/EXAMPLE_CONFIG.md](EXAMPLE_CONFIG.md) for detailed configuration examples and templates.

## 驾驶数据生成 (Driving Data Generation)

### 核心组件 (Core Components)

#### 1. 数据收集脚本
- **主脚本**: `collect_dataset_slurm.py`
- **SLURM 启动脚本**: `0_run_collect_dataset_slurm.sh`
- **专家代理**: PDM-Lite from DriveLM

#### 2. 关键配置参数

```python
# 在 collect_dataset_slurm.py 中需要修改的路径 (lines 213-230)
code_root = "/path/to/simlingo"           # SimLingo 代码根目录
carla_root = "/path/to/CARLA/root"        # CARLA 安装路径
dataset_name = "simlingo_v2_YYYY_MM_DD"   # 数据集名称
root_folder = "database/"                 # 数据存储根目录
```

#### 3. SLURM 配置文件

```bash
# partition.txt - 指定 SLURM 分区
YOUR_PARTITION

# max_num_jobs.txt - 并行作业数量 (建议从小数字开始测试)
10
```

### 路由文件管理 (Route File Management)

#### 路由文件生成
```bash
# 从原始 CARLA 路由文件生成修改版本
bash dataset_generation/split_route_files.sh
```

**路由文件特点:**
- 存储位置: `data/simlingo/`
- 短路由: 最多 1-3 个场景
- 平衡采样: 所有场景类型均匀分布
- 上采样: 稀有场景增加采样频率

#### 路由文件结构
```
data/simlingo/
├── training/
│   ├── routes_training_balanced/     # 平衡采样的训练路由
│   └── routes_training_lb1/          # Leaderboard 1.0 格式路由
└── validation/
    ├── routes_validation_balanced/
    └── routes_validation_lb1/
```

### 数据收集流程 (Data Collection Workflow)

#### 1. 环境设置
```bash
export CARLA_ROOT=/path/to/CARLA/root
export WORK_DIR=/path/to/simlingo
export PYTHONPATH=$PYTHONPATH:${CARLA_ROOT}/PythonAPI/carla
export SCENARIO_RUNNER_ROOT=${WORK_DIR}/scenario_runner_autopilot
export LEADERBOARD_ROOT=${WORK_DIR}/leaderboard_autopilot
```

#### 2. 启动数据收集
```bash
# 提交 SLURM 作业
sbatch 0_run_collect_dataset_slurm.sh

# 监控作业状态
squeue -u YOUR_USERNAME

# 调整并行作业数 (运行时修改)
echo "20" > max_num_jobs.txt
```

#### 3. 监控和管理
- 脚本自动监控崩溃的进程
- 自动重启失败的路由
- 实时调整并行作业数量
- 生成详细的日志文件

### 数据清理 (Dataset Cleaning)

#### 清理失败的运行
```bash
# 删除专家失败的路由
python dataset_generation/delete_failed_runs.py

# 删除有违规的路由
python dataset_generation/delete_infraction_routes.py

# 删除重复路由
python dataset_generation/filter_duplicate_routes.py

# 删除损坏的 JSON 文件
python dataset_generation/delete_defect_jsons.py
```

### 输出数据结构 (Output Data Structure)

每个成功收集的路由包含以下文件：
```
database/simlingo_v2_YYYY_MM_DD/
└── route_XXXX/
    ├── rgb_front/          # 前视摄像头图像
    ├── lidar/             # 激光雷达点云
    ├── measurements/      # 车辆状态和环境信息
    ├── route_XXXX.xml     # 路由定义文件
    └── scenario_log.json  # 场景执行日志
```

#### 关键数据字段
```json
{
  "measurements": {
    "ego_vehicle": {
      "speed": "float - 车辆速度 (m/s)",
      "acceleration": "float - 加速度 (m/s²)",
      "location": "[x, y, z] - 世界坐标",
      "rotation": "[pitch, yaw, roll] - 旋转角度"
    },
    "route": "list - 专家路径点",
    "traffic_lights": "list - 交通信号灯状态",
    "other_vehicles": "list - 其他车辆信息",
    "walkers": "list - 行人信息",
    "scenario_info": "dict - 当前场景信息"
  }
}
```

## 语言数据生成 (Language Data Generation)

### VQA 数据生成 (DriveLM)

#### 概述
基于 DriveLM 框架生成驾驶场景的问答数据，包含感知、预测、规划三个层次的问题。

#### 运行脚本
```bash
python dataset_generation/language_labels/drivelm/carla_vqa_generator_main.py \
    --data-directory database/simlingo \
    --output-directory database/simlingo/drivelm \
    --sample-frame-mode all \
    --save-examples
```

#### 参数说明
- `--data-directory`: 驾驶数据所在目录
- `--output-directory`: VQA 数据输出目录
- `--sample-frame-mode`: 采样模式 ['all', 'keyframes', 'uniform']
- `--target-image-size`: 目标图像尺寸 [width, height]
- `--save-examples`: 保存可视化示例

#### VQA 数据结构
```json
{
  "image": "rgb_front/frame_XXXXX.jpg",
  "questions": [
    {
      "question": "What objects are in the scene?",
      "answer": "There are cars, traffic lights, and pedestrians.",
      "question_type": "perception",
      "objects": ["car", "traffic_light", "pedestrian"],
      "template": "perception_objects"
    }
  ],
  "scene_info": {
    "town": "Town01",
    "weather": "clear",
    "time_of_day": "noon"
  }
}
```

#### 问题类型 (Question Types)
1. **感知层 (Perception)**: 物体检测、场景理解
2. **预测层 (Prediction)**: 行为预测、轨迹预测  
3. **规划层 (Planning)**: 决策解释、动作合理性

### 解说数据生成 (Commentary)

#### 概述
生成对驾驶动作和决策的自然语言解说，解释专家为什么做出特定的驾驶行为。

#### 运行脚本
```bash
python dataset_generation/language_labels/commentary/carla_commentary_generator_main.py \
    --data-directory database/simlingo \
    --output-directory database/simlingo/commentary \
    --sample-frame-mode all
```

#### 解说数据结构
```json
{
  "image": "rgb_front/frame_XXXXX.jpg",
  "commentary": "I am slowing down because there is a red traffic light ahead.",
  "commentary_template": "I am {action} because there is a {object_color} {object_type} {location}.",
  "cause_object_visible_in_image": true,
  "cause_object": {
    "type": "traffic_light",
    "color": "red",
    "location": [x, y, z],
    "distance": 25.5
  },
  "cause_object_string": "red traffic light that is ahead",
  "scenario_name": "TrafficLightScenario",
  "placeholder": {
    "action": "slowing down",
    "object_color": "red", 
    "object_type": "traffic light",
    "location": "ahead"
  }
}
```

#### 解说类型 (Commentary Types)
1. **速度调整**: 加速、减速、停车
2. **转向决策**: 左转、右转、直行
3. **车道变更**: 变道、超车、合并
4. **交通规则**: 交通信号、停车标志、让行

### 增强模板 (Augmented Templates)

#### VQA 增强模板
位置: `data/augmented_templates/drivelm_train_augmented_v2/`
```
├── all_objs_augmented.json     # 物体相关问题增强
├── all_as_augmented.json       # 动作相关问题增强
└── ...
```

#### 解说增强模板  
位置: `data/augmented_templates/commentary_augmented.json`

#### 增强方法
使用 ChatGPT 生成同义词、短语变体和句式重组：
```bash
python dataset_generation/get_augmentations/gpt_augment_vqa.py
python dataset_generation/get_augmentations/commentary_merge_augmented.py
```

## 梦想数据生成 (Dreamer Data Generation)

### 概述
Action Dreaming 为给定的语言指令生成多种可能的未来轨迹，用于改善语言-动作对齐。

### 运行脚本
```bash
python dataset_generation/dreamer_data/dreamer_generator.py
```

### 核心功能

#### 1. 轨迹生成类型
- **速度变化**: 加速、减速、目标速度
- **车道变更**: 左变道、右变道
- **物体导航**: 朝向特定物体、避开障碍物
- **碰撞场景**: 不安全的轨迹生成

#### 2. 安全评估
每个生成的轨迹都包含安全标签：
- `allowed`: 是否允许执行
- `safe_to_execute`: 是否安全执行
- `dreamer_answer_safety`: 安全模式下的回答

### 梦想数据结构
```json
{
  "category": "target_speed",
  "waypoints": [[x1, y1], [x2, y2], ...],
  "route": [[x1, y1], [x2, y2], ...],
  "rgb_path": "rgb_front/frame_XXXXX.jpg",
  "allowed": true,
  "mode": "target_speed",
  "info": {
    "current_speed": 8.5,
    "target_speed": 15.0,
    "final_speed": 14.8,
    "acceleration_needed": 2.1
  },
  "route_reasoning": "The route maintains the current lane while gradually accelerating to reach the target speed.",
  "dreamer_instruction": "Accelerate to 15 m/s while staying in the current lane.",
  "instructions_templates": "Accelerate to {target_speed} m/s while staying in the {lane_position} lane.",
  "templates_placeholders": {
    "target_speed": "15",
    "lane_position": "current"
  },
  "dreamer_answer_safety": "This instruction is safe to execute as there are no obstacles ahead and the target speed is within the speed limit.",
  "safe_to_execute": true
}
```

### 指令类别 (Instruction Categories)

1. **速度控制**
   - `target_speed`: 达到目标速度
   - `faster`: 加速
   - `slower`: 减速
   - `stop`: 停车

2. **方向控制**
   - `left_lane_change`: 左变道
   - `right_lane_change`: 右变道
   - `turn_left`: 左转
   - `turn_right`: 右转

3. **物体交互**
   - `follow_vehicle`: 跟随车辆
   - `overtake`: 超车
   - `avoid_obstacle`: 避开障碍

4. **特殊场景**
   - `crash`: 碰撞场景（用于负例学习）
   - `emergency_stop`: 紧急停车

## 数据分桶 (Data Buckets)

### 概述
数据分桶用于组织和分析收集的数据，提供数据集的统计信息和质量评估。

### 生成分桶信息
```bash
python dataset_generation/data_buckets/carla_get_buckets.py
```

### 分桶统计
```bash
python dataset_generation/data_buckets/get_bucket_stats.py
python dataset_generation/data_buckets/bucket_size_stats.py
```

### 分桶结构
数据按以下维度分桶：
- **场景类型**: 交叉口、高速公路、城市道路
- **天气条件**: 晴天、雨天、雾天
- **时间**: 白天、黄昏、夜间
- **交通密度**: 低、中、高
- **驾驶行为**: 直行、转弯、变道、停车

## 数据增强 (Data Augmentation)

### ChatGPT 增强

#### VQA 增强
```bash
python dataset_generation/get_augmentations/gpt_augment_vqa.py \
    --input-templates data/templates/drivelm_original.json \
    --output-file data/augmented_templates/drivelm_train_augmented_v2/
```

#### 解说增强
```bash
python dataset_generation/get_augmentations/commentary_merge_augmented.py \
    --subsentence-file data/augmented_templates/commentary_subsentence.json \
    --output-file data/augmented_templates/commentary_augmented.json
```

### 增强策略
1. **同义词替换**: 使用语义相似的词汇
2. **句式变换**: 改变句子结构但保持语义
3. **详细程度调整**: 增加或减少描述细节
4. **语言风格变化**: 正式/非正式、技术/日常用语

## 故障排除 (Troubleshooting)

### 常见问题

#### 1. CARLA 连接问题
```bash
# 检查 CARLA 服务是否运行
ps aux | grep CarlaUE4

# 检查端口占用
netstat -tulpn | grep 2000

# 重启 CARLA 服务
killall CarlaUE4.sh
${CARLA_ROOT}/CarlaUE4.sh -RenderOffScreen
```

#### 2. SLURM 作业失败
```bash
# 查看作业日志
cat slurm/logs/collect_JOBID.out
cat slurm/logs/collect_JOBID.err

# 检查资源使用
scontrol show job JOBID

# 重新提交失败的作业
sbatch --dependency=afternotok:FAILED_JOBID 0_run_collect_dataset_slurm.sh
```

#### 3. 内存不足
- 减少并行作业数: `echo "5" > max_num_jobs.txt`
- 增加 SLURM 内存限制: `#SBATCH --mem=16G`
- 使用数据流处理而非批量加载

#### 4. 磁盘空间不足
```bash
# 检查磁盘使用情况
df -h

# 清理临时文件
find database/ -name "*.tmp" -delete

# 压缩旧数据
tar -czf old_data.tar.gz database/old_dataset/
```

### 性能优化

#### 1. 并行化建议
- GPU 数量 = SLURM 并行作业数
- 每个 GPU 运行一个 CARLA 实例
- 监控 GPU 内存使用率

#### 2. 存储优化
- 使用 SSD 存储活跃数据
- 定期压缩完成的数据
- 使用网络存储备份

#### 3. 网络优化
- 减少不必要的数据传输
- 使用本地缓存
- 压缩传输数据

## 最佳实践 (Best Practices)

### 数据收集

1. **逐步扩展**: 从少量路由开始测试，确认无误后再大规模运行
2. **监控质量**: 定期检查生成数据的质量和完整性
3. **备份策略**: 设置自动备份和版本控制
4. **资源管理**: 合理分配计算资源，避免系统过载

### 代码管理

1. **版本控制**: 为不同版本的数据生成代码打标签
2. **配置分离**: 将配置参数从代码中分离出来
3. **日志记录**: 详细记录每次数据生成的参数和结果
4. **单元测试**: 为关键函数编写测试用例

### 数据质量

1. **验证脚本**: 编写自动化脚本验证数据完整性
2. **统计分析**: 定期分析数据分布和质量指标
3. **人工抽查**: 随机抽取样本进行人工质量检查
4. **增量更新**: 支持增量添加新数据而不影响现有数据

### 扩展性考虑

1. **模块化设计**: 保持各组件的独立性和可替换性
2. **配置灵活性**: 支持不同场景和需求的配置调整
3. **文档维护**: 及时更新文档以反映代码变更
4. **社区贡献**: 鼓励社区贡献和改进建议

## 总结 (Summary)

SimLingo 的数据生成管道是一个复杂但模块化的系统，包含：

1. **驾驶数据收集**: 使用 PDM-Lite 在 CARLA 中收集高质量驾驶数据
2. **语言标签生成**: 生成 VQA 和解说数据以支持语言理解
3. **梦想数据创建**: 通过 Action Dreaming 改善语言-动作对齐
4. **数据增强**: 使用 ChatGPT 增加数据多样性

通过遵循本文档的指导，您应该能够成功复现和扩展 SimLingo 的数据生成流程。如有问题，请参考故障排除部分或提交 Issue。