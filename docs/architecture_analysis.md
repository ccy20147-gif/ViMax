# ViMax 项目架构解析文档

> AI视频生成工作流系统 - 从创意/剧本/小说自动到视频

---

## 一、项目概览

ViMax 是一个端到端的 AI 视频生成系统，支持多种输入源（创意、剧本、小说），通过多个智能体协作，自动生成具有电影级镜头语言的视频内容。

### 核心能力
- **创意到视频**：一句话创意 → 完整故事 → 场景剧本 → 分镜 → 视频
- **小说到电影**：长篇小说 → 事件提取 → 场景分解 → 视频
- **剧本到视频**：单场景剧本 → 分镜设计 → 视频

---

## 二、核心 Pipeline 架构

```
┌─────────────────────────────────────────────────────────────┐
│                      用户输入层                              │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐              │
│  │  Idea    │    │  Novel   │    │  Script  │              │
│  └────┬─────┘    └────┬─────┘    └────┬─────┘              │
└───────┼──────────────┼──────────────┼──────────────────────┘
        │              │              │
        ▼              ▼              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Pipeline 层                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │ Idea2Video   │  │ Novel2Movie  │  │ Script2Video │     │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘     │
└─────────┼────────────────┼────────────────┼────────────────┘
          │                │                │
          └────────────────┴────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                     Agent 协作层                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ Screenwriter │  │ Storyboard   │  │ Character    │      │
│  │   (编剧)     │  │   Artist     │  │  Extractor   │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                     生成层                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ Image Gen    │  │  Video Gen   │  │  Audio Gen   │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
```

---

## 三、剧本生成流程

### 3.1 Screenwriter Agent（编剧智能体）

**职责**：将创意/想法转化为完整的剧本

#### 核心方法

1. **develop_story(idea, user_requirement)**
   - 输入：一句话创意 + 用户需求（可选）
   - 输出：完整的故事文本
   - Prompt 要求：
     - 三幕式/英雄之旅等经典叙事结构
     - 立体角色（动机、缺陷、成长弧线）
     - 视觉化思维（场景氛围、关键动作、对白）
     - Show Don't Tell 原则

2. **write_script_based_on_story(story, user_requirement)**
   - 输入：完整故事 + 用户需求
   - 输出：场景列表 `List[str]`
   - 特点：
     - 按时间和地点连续性分割场景
     - 标准剧本格式（场景标题、角色名、对白、动作描述）
     - 视觉增强原则（所有描述必须"可拍摄"）

#### 输出示例
```python
# 每个场景是纯文本剧本
scene_scripts = [
    "Scene 1: ...\n<Alice>: Hello...\n<Alice> turns away...",
    "Scene 2: ...\n<Bob> enters...",
]
```

---

## 四、镜头控制机制

### 4.1 Camera 数据结构

```python
class Camera(BaseModel):
    idx: int                                    # 镜头序号
    active_shot_idxs: List[int]                 # 该镜头拍摄的 shot 列表
    parent_cam_idx: Optional[int]               # 父镜头索引
    parent_shot_idx: Optional[int]              # 父 shot 索引
    is_parent_fully_covers_child: Optional[bool] # 父镜头是否完全覆盖子镜头
    missing_info: Optional[str]                 # 子镜头中未被父镜头覆盖的信息
```

**关键概念**：
- **Camera**：物理摄像机位置，可拍摄多个 Shot
- **Shot**：一个连续的镜头片段
- **Camera Tree**：镜头之间的层级关系树

### 4.2 镜头树构建流程

```
Shot 0 (Wide) ──────────────────────────────┐
    │                                       │
    ├── Shot 1 (Medium, Same Camera)        │
    │                                       │
    └── Shot 2 (Close-up, New Camera) ──────┼─── Camera Tree
           │                                │
           └── Shot 3 (Another Angle)       │
```

**构建策略**：
1. 分析所有 Shot 的 `cam_idx`（镜头位置）
2. 识别同一位置的多个 Shot → 归为同一 Camera
3. 调用 `camera_image_generator.construct_camera_tree()` 建立父子关系
4. 判断 `is_parent_fully_covers_child` → 决定是否需要生成新的参考图

### 4.3 ShotDescription（镜头描述）

```python
class ShotDescription(BaseModel):
    idx: int                          # shot 序号
    is_last: bool                     # 是否最后一个 shot
    cam_idx: int                      # 使用的镜头索引
    visual_desc: str                  # 整体视觉描述
    
    # 首末帧分解
    ff_desc: str                      # First Frame 描述
    ff_vis_char_idxs: List[int]       # 首帧可见角色
    lf_desc: str                      # Last Frame 描述
    lf_vis_char_idxs: List[int]       # 末帧可见角色
    motion_desc: str                  # 运动描述（相机+物体）
    
    # 变化类型
    variation_type: Literal["large", "medium", "small"]
    variation_reason: str
    
    audio_desc: str                   # 音频描述
```

**变化类型定义**：
| 类型 | 描述 | 典型场景 |
|------|------|----------|
| **large** | 首末帧构图和焦点显著变化 | 航拍过渡、远景→特写 |
| **medium** | 新角色出现或角色转身 | 角色加入、背面→正面 |
| **small** | 微小变化 | 表情变化、轻微移动、平移 |

### 4.4 StoryboardArtist Agent（分镜艺术家）

**核心能力**：
1. **design_storyboard()** - 从剧本生成分镜概览
2. **decompose_visual_description()** - 将视觉描述分解为三部分

**分解逻辑**：
```
输入: "中景镜头，Alice 和 Bob 在超市..."
       ↓
输出: {
    ff_desc: "首帧：Alice 在左，Bob 在右...",
    lf_desc: "末帧：Bob 转身面对 Alice...",
    motion_desc: "Bob 缓慢转身，表情从惊讶变为喜悦...",
    variation_type: "medium"
}
```

**镜头语言指导**：
- 使用专业电影术语（dolly, pan, tilt, track...）
- 首帧必须建立整体环境（最宽镜头）
- 尽可能少用镜头位置
- 每个 shot 独立描述，不互相引用

---

## 五、一致性问题解决方案

### 5.1 多层级角色体系

```
Novel Level (全局)
    │
    ├── CharacterInNovel: {
    │       index, identifier_in_novel,
    │       active_events: {0: "Alice", 2: "Alice in Wonderland"},
    │       static_features: "长发、蓝眼睛..."
    │   }
    │
    ▼
Event Level (事件级)
    │
    ├── CharacterInEvent: {
    │       index, identifier_in_event,
    │       active_scenes: {0: "Alice", 2: "Alice"},
    │       static_features: "合并后的特征..."
    │   }
    │
    ▼
Scene Level (场景级)
    │
    └── CharacterInScene: {
            idx, identifier_in_scene,
            is_visible: True/False,
            static_features: "固定特征（面部、体型）",
            dynamic_features: "动态特征（服装、配饰）"
        }
```

### 5.2 角色肖像系统

**三级肖像生成**：

1. **基础肖像**（Base Portrait）
   - 基于 `static_features` 生成
   - 白色背景，正面全身照
   - 作为后续所有变体的基础

2. **多角度肖像**
   - Front（正面）- 主参考
   - Side（侧面）- 辅助参考
   - Back（背面）- 辅助参考

3. **场景级肖像**（Scene Portrait）
   - 基于 `dynamic_features` 调整
   - 保持角色身份一致性
   - 只改变服装/配饰

**存储结构**：
```
character_portraits/
├── base/
│   ├── character_0_Alice.png
│   └── character_1_Bob.png
├── 0_Alice/
│   ├── front.png
│   ├── side.png
│   └── back.png
├── 1_Bob/
│   ├── front.png
│   ├── side.png
│   └── back.png
└── event_0/
    └── scene_0/
        └── character_0_Alice.png  # 带动态特征
```

### 5.3 参考图选择机制

**ReferenceImageSelector Agent**：

```python
async def select_reference_images_and_generate_prompt(
    available_image_path_and_text_pairs: List[Tuple[str, str]],
    frame_description: str
) -> dict:
    # 返回
    {
        "reference_image_path_and_text_pairs": [...],  # 选中的参考图
        "text_prompt": "..."                           # 生成提示词
    }
```

**选择策略**：
1. 收集所有可用参考图：
   - 角色肖像（正面/侧面/背面）
   - 首帧参考图（来自父镜头）
   - 父镜头过渡帧
2. 根据目标帧描述，智能选择最相关的参考图
3. 生成包含参考图说明的组合 prompt

### 5.4 镜头间一致性保障

**Camera Tree 父子关系**：
```
Camera 0 (Wide Shot)
    ├── Shot 0 (First Shot)
    │   └── 生成 first_frame
    │
    └── Camera 1 (Close-up)
        ├── Shot 1
        │   ├── 等待 Shot 0 的 first_frame
        │   ├── 生成 transition_video（过渡视频）
        │   ├── 从过渡视频提取 new_camera_image
        │   └── 使用父镜头参考 + 角色肖像生成 first_frame
        │
        └── Shot 2
            └── 使用 Camera 1 的 first_frame 作为参考
```

**过渡视频机制**：
1. 等待父镜头的 first_frame 生成完成
2. 生成过渡视频（从父镜头到子镜头）
3. 从过渡视频中提取关键帧作为参考
4. 使用参考图 + 角色肖像生成子镜头的 first_frame

---

## 六、完整工作流详解

### 6.1 Idea2Video Pipeline（创意到视频）

```
Step 1: develop_story(idea)
   └── Screenwriter 生成完整故事

Step 2: extract_characters(story)
   └── CharacterExtractor 提取角色列表

Step 3: generate_character_portraits(characters)
   └── 为每个角色生成正面/侧面/背面肖像

Step 4: write_script_based_on_story(story)
   └── 将故事改编为场景剧本列表

Step 5: 对每个场景执行 Script2VideoPipeline
   └── 场景视频列表

Step 6: concatenate_videos()
   └── 拼接所有场景视频
```

### 6.2 Script2Video Pipeline（剧本到视频）

```
Input: 单场景剧本 + 角色列表 + 肖像注册表

Step 1: extract_characters(script)
   └── 提取/验证场景角色

Step 2: generate_character_portraits(characters)
   └── 生成角色肖像（如果没有）

Step 3: design_storyboard(script, characters)
   └── StoryboardArtist 设计分镜
   └── 输出: List[ShotBriefDescription]

Step 4: decompose_visual_descriptions(storyboard)
   └── 将每个 Shot 的视觉描述分解
   └── 输出: List[ShotDescription] (含首帧/末帧/运动)

Step 5: construct_camera_tree(shot_descriptions)
   └── 构建镜头树
   └── 输出: List[Camera] (含父子关系)

Step 6: generate_frames_for_each_camera()
   └── 并行处理每个 Camera
   └── 对每个 Camera:
       ├── 生成第一个 Shot 的 first_frame
       ├── 如果是子镜头:
       │   ├── 等待父镜头 first_frame
       │   ├── 生成过渡视频
       │   └── 提取参考图
       ├── 选择参考图 + 生成 prompt
       └── 生成 first_frame/last_frame

Step 7: generate_video_for_each_shot()
   └── 对每个 Shot:
       ├── 等待 first_frame
       ├── 等待 last_frame (if variation != small)
       └── 调用 video_generator 生成视频

Step 8: concatenate_videos()
   └── 拼接所有 Shot 视频
```

### 6.3 Novel2Movie Pipeline（小说到电影）

```
Step 1: 小说压缩
   ├── 分块 (chunk_size=512)
   ├── 并行压缩每个块 (semaphore=5)
   └── 合并压缩结果

Step 2: 事件提取
   ├── 逐个提取事件 (is_last 控制结束)
   └── 每个事件包含 process_chain（因果链）

Step 3: 知识库检索
   ├── 构建 FAISS 向量库
   ├── 对每个事件的 process 步骤检索相关块
   ├── Reranker 重排序 (threshold=0.7)
   └── 合并得分

Step 4: 场景提取
   ├── 对每个事件提取场景
   └── 每个场景包含 characters + environment + script

Step 5: 角色合并
   ├── Scene Level → Event Level
   └── Event Level → Novel Level

Step 6: 角色肖像生成
   ├── 基础肖像 (static_features)
   └── 场景级肖像 (dynamic_features)

Step 7: 视频生成
   └── 对每个 event × scene 执行 Script2VideoPipeline
```

---

## 七、关键技术细节

### 7.1 并发控制

```python
# 使用 asyncio.Semaphore 控制并发
sem = asyncio.Semaphore(5)  # 最多5个并发

async def task(sem, ...):
    async with sem:
        # 执行任务
        pass

# 并行执行但限制并发
tasks = [task(sem, ...) for item in items]
await asyncio.gather(*tasks)
```

**并发限制**：
- 小说压缩：5
- 知识库检索：10
- 场景提取：8
- 角色肖像生成：5
- 事件级角色合并：8

### 7.2 缓存机制

**文件级缓存**：
- 每个步骤的结果都会保存到 JSON/TXT 文件
- 下次运行时跳过已存在的文件
- 支持断点续传

```python
if os.path.exists(save_path):
    print(f"🚀 Loaded from existing file.")
    return cached_data
else:
    result = await generate(...)
    save_to_file(result, save_path)
    return result
```

### 7.3 事件驱动的异步协调

```python
# 使用 asyncio.Event 协调依赖
self.frame_events = {
    shot_idx: {
        "first_frame": asyncio.Event(),
        "last_frame": asyncio.Event()
    }
}

# 等待依赖完成
await self.frame_events[parent_shot_idx]["first_frame"].wait()

# 标记完成
self.frame_events[shot_idx]["first_frame"].set()
```

### 7.4 Pydantic 数据验证

所有接口使用 Pydantic BaseModel：
- 自动 JSON 序列化/反序列化
- 字段验证和类型检查
- 自动生成 OpenAI function calling schema

---

## 八、Agent Prompt 设计要点

### 8.1 Screenwriter Prompt
- 角色：创意故事生成专家
- 要求：三幕结构、立体角色、Show Don't Tell
- 输出：结构化故事文档

### 8.2 StoryboardArtist Prompt
- 角色：专业分镜艺术家
- 要求：
  - 首帧建立环境（最宽镜头）
  - 尽可能少用镜头位置
  - 每个 shot 独立描述
  - 标注角色朝向和位置

### 8.3 Visual Description Decomposition Prompt
- 角色：视觉文本分析师
- 输出：首帧/末帧/运动 三部分
- 关键：
  - 首末帧是"快照"，不含进行中的动作
  - 运动描述用专业电影术语
  - 角色用外观特征而非名字

---

## 九、待优化项（代码中的 TODO）

1. **角色查找效率**：
   ```python
   # TODO: 这里的数据结构没有做好，居然还要遍历查找
   character_in_event = [char for char in characters_in_event if ...][0]
   ```

2. **数据结构层级**：
   - Novel → Event → Scene 的角色映射需要优化
   - 考虑使用字典而非列表

3. **Novel2Movie Pipeline 未完成**：
   ```python
   # TODO: NOT IMPLEMENTED YET
   ```

---

## 十、总结

ViMax 的核心创新点：

1. **镜头树机制**：通过父子关系管理镜头过渡，确保视觉连贯性
2. **三级角色体系**：Novel/Event/Scene 层级管理角色一致性
3. **首末帧分解**：将连续动作分解为静态帧，便于生成
4. **参考图选择**：智能选择最相关的参考图组合
5. **过渡视频**：在不同镜头间生成平滑过渡

这套架构使得 AI 生成的视频具有专业电影级的镜头语言和视觉连贯性。
