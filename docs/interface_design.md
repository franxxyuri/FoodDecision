# FoodDecision 接口设计文档

## 1. 概述
- **目标**：为“帮我决定今天吃什么”的全 Compose 安卓应用提供模块化、低耦合、易调试的接口方案。
- **核心能力**：多数据源聚合、随机/权重筛选、用户偏好管理、收藏与历史记录。
- **依赖方向**：`:app → :feature:* → :core:*`，严格自上而下。

## 2. 模块与关键接口

| 模块 | 描述 | 主要接口 |
| --- | --- | --- |
| `:app` | Activity / Nav host、DI 入口 | `FoodDecisionNavGraph`, `FeatureEntry` |
| `:core:model` | 纯数据模型 | `FoodItem`, `FoodFilters`, `FoodSuggestion`, `SelectionContext` |
| `:core:common` | 基础工具、Result、Logger、Dispatcher | `Result<T>`, `DispatcherProvider`, `Logger` |
| `:core:designsystem` | Compose Theme、组件库 | `FoodDecisionTheme`, `FDButton`, `SectionCard` |
| `:core:data` | Repository 接口 + Selection 管道 | `FoodRepository`, `PreferenceRepository`, `HistoryRepository`, `SelectionStrategy`, `SelectionPipeline` |
| `:data:datasource:*` | 具体数据源实现 | `FoodDataSource`（Local / Remote / Community 等实现） |
| `:feature:decision` | 今日推荐界面 | `DecisionRoute`, `DecisionViewModel`, `DecisionUiState`, `DecisionEvent` |
| `:feature:favorites` | 收藏功能 | `FavoritesRoute`, `FavoritesViewModel` |
| `:feature:preferences` | 偏好设置 | `PreferencesRoute`, `PreferencesViewModel` |

## 3. 数据模型（`core:model`）
```kotlin
data class FoodItem(
    val id: String,
    val name: String,
    val tags: List<FoodTag>,
    val nutrition: Nutrition,
    val priceRange: PriceRange,
    val origin: FoodSource
)

data class FoodFilters(
    val mealType: MealType?,
    val tagsInclude: Set<FoodTag>,
    val tagsExclude: Set<FoodTag>,
    val maxCalories: Int?,
    val budget: PriceRange?,
    val cookingTime: Int?,
    val allergies: Set<Allergy>
)

data class FoodSuggestion(
    val item: FoodItem,
    val source: FoodSource,
    val strategy: StrategyType,
    val score: Double,
    val reason: String
)

data class SelectionContext(
    val recentDecisions: List<FoodItem>,
    val userPreferences: UserPreferences,
    val timestamp: Instant,
    val environment: DecisionEnvironment // 网络状态、定位等
)
```

## 4. Repository 与数据源接口（`core:data` & `data:datasource`）
```kotlin
interface FoodRepository {
    suspend fun getSuggestion(filters: FoodFilters): Result<FoodSuggestion>
    suspend fun getPool(filters: FoodFilters): Result<List<FoodItem>>
}

interface PreferenceRepository {
    suspend fun getPreferences(): UserPreferences
    suspend fun updatePreferences(prefs: UserPreferences)
    suspend fun toggleFavorite(itemId: String): Result<Unit>
    fun observeFavorites(): Flow<List<FoodItem>>
}

interface HistoryRepository {
    suspend fun logDecision(suggestion: FoodSuggestion, action: DecisionAction)
    fun observeHistory(limit: Int): Flow<List<DecisionLog>>
}
```

### 数据源抽象
```kotlin
interface FoodDataSource {
    val id: DataSourceId
    val priority: Int
    suspend fun fetchFoods(filters: FoodFilters): Result<List<FoodItem>>
    fun supports(filters: FoodFilters): Boolean
}
```
- 通过 Hilt multi-binding 注入 `Set<FoodDataSource>`。
- `priority` 用于策略化调度（如离线优先）。
- `supports` 决定数据源是否能满足本次请求（如远程源可能不支持极窄过滤器）。

## 5. 选择策略与管线（`core:algorithm` 包）
```kotlin
interface SelectionStrategy {
    fun pick(
        candidates: List<ScoredFoodItem>,
        filters: FoodFilters,
        context: SelectionContext
    ): FoodSuggestion
}

interface SelectionPipeline {
    suspend fun buildPool(
        filters: FoodFilters,
        context: SelectionContext
    ): Result<List<ScoredFoodItem>>
}
```
- `SelectionPipeline` 流程：
  1. **SourceStage**：并行请求各 `FoodDataSource`
  2. **FilterStage**：食材禁忌、预算、时间等硬过滤
  3. **ScoreStage**：根据偏好 / 营养 / 新鲜度打分，产出 `ScoredFoodItem`
  4. **DiversityStage**：结合 `recentDecisions` 降低重复权重
- `SelectionStrategy` 决定最终挑选（纯随机 / 加权 / 混合等）。
- `StrategyType` 示例：`PURE_RANDOM`, `WEIGHTED_RANDOM`, `RULE_BASED`, `ML_RANKING`。

## 6. Feature 层接口（以 Decision 为例）
```kotlin
interface DecisionEntry : FeatureEntry {
    @Composable
    fun NavGraph(startDestination: String)
}

data class DecisionUiState(
    val currentSuggestion: FoodSuggestion?,
    val filters: FoodFilters,
    val isLoading: Boolean,
    val error: UiError?,
    val history: List<FoodSuggestion>
)

sealed interface DecisionEvent {
    data object Refresh : DecisionEvent
    data class FiltersChanged(val filters: FoodFilters) : DecisionEvent
    data class ConfirmSelection(val suggestion: FoodSuggestion) : DecisionEvent
    data class ToggleFavorite(val itemId: String) : DecisionEvent
}
```
- `DecisionViewModel` 依赖 `FoodRepository`, `PreferenceRepository`, `HistoryRepository`, `SelectionStrategyResolver`。
- Compose UI 只消费 `DecisionUiState`，通过 `DecisionEvent` 与 ViewModel 交互。

## 7. 调试与扩展接口
- `SelectionTraceLogger.log(trace: SelectionTrace)`：记录候选、权重、落选原因，供 Debug 页或日志使用。
- `FeatureToggle`（`core:common`）：
  ```kotlin
  interface FeatureToggle {
      fun isEnabled(flag: ToggleFlag): Boolean
  }
  ```
  控制策略 / 数据源启用，便于灰度与调试。
- `StrategyResolver`：根据用户选择或服务器配置返回 `SelectionStrategy` 实例。

## 8. 接口协作示例
1. `DecisionViewModel` 收到 `Refresh`：
   1. `PreferenceRepository.getPreferences()` 获取上下文
   2. `FoodRepository.getSuggestion(filters)` → 内部通过 `SelectionPipeline + Strategy`
   3. 成功：更新 `DecisionUiState.currentSuggestion`，并 `HistoryRepository.logDecision`
   4. 失败：`UiState.error = UiError(message, traceId)`

## 9. 未来扩展
- 新增数据源：实现 `FoodDataSource` 并注册 Hilt binding，其他层无需修改。
- 新策略：实现 `SelectionStrategy` 并通过 `StrategyResolver` 暴露。
- 统计 / 推荐服务：在 `ScoreStage` 接入远程排序 API，UI 与 ViewModel 保持不变。
