# Task B1: MCP-Reasoner Integration Setup

**Task ID**: task-381  
**GitHub Issue**: #5 (Flow Intelligence Enhancement)  
**Category**: B - Enhanced MCP Integration  
**Priority**: IMMEDIATE (Next Task)  
**Estimated Time**: 2-3 hours  

## Overview

Integrate the `mcp-reasoner` MCP server into FLUJO's core FlowExecutor to enable intelligent flow analysis, optimization suggestions, and dynamic decision-making capabilities. This is the foundational task for all chain of thought management improvements.

## Strategic Importance

This task establishes the **reasoning foundation** for the entire FLUJO intelligence enhancement project. All subsequent intelligent features depend on this implementation:

- **Dynamic Branching** (Task B4) requires reasoning analysis
- **Memory Integration** (Task C1-C2) needs reasoning for pattern recognition  
- **Flow Generation** (Task D1-D3) depends on reasoning for optimization
- **Analytics** (Task E1-E3) uses reasoning for insights

## Technical Specifications

### Implementation Location
- **Primary File**: `src/backend/execution/flow/FlowExecutor.ts`
- **New Service**: `src/backend/services/mcp/reasoner-integration.ts`
- **Dependencies**: mcp-reasoner MCP server, MCPHandler

### Core Implementation

#### 1. FlowExecutor Enhancement
```typescript
// Add to existing FlowExecutor class
export class FlowExecutor {
  
  // NEW METHOD: Pre-execution analysis
  private static async analyzeFlowOptimization(
    sharedState: SharedState
  ): Promise<FlowOptimizationResult> {
    const reasoningResult = await MCPHandler.executeMCP({
      mcpServer: 'mcp-reasoner',
      enabledTools: ['mcp-reasoner'],
      context: {
        messages: sharedState.messages,
        currentFlow: sharedState.flowId,
        executionHistory: sharedState.debugSteps,
        performanceMetrics: sharedState.metrics || {},
        availableNodes: sharedState.nodes || []
      },
      task: 'analyze_flow_performance_and_suggest_optimizations'
    });
    
    return {
      suggestions: reasoningResult.value.suggestions || [],
      alternativePaths: reasoningResult.value.alternatives || [],
      performanceMetrics: reasoningResult.value.metrics || {},
      confidence: reasoningResult.value.confidence || 0.5,
      reasoning: reasoningResult.value.reasoning || 'No specific reasoning provided'
    };
  }
  
  // MODIFIED METHOD: Enhanced executeStep with reasoning
  static async executeStep(
    stepId: string, 
    sharedState: SharedState
  ): Promise<ExecutionResult> {
    
    // NEW: Pre-execution reasoning analysis
    const optimization = await this.analyzeFlowOptimization(sharedState);
    
    // Add optimization insights to shared state
    sharedState.optimizationInsights = optimization;
    
    // Continue with existing execution logic...
    const step = sharedState.debugSteps.find(s => s.stepId === stepId);
    // ... rest of existing implementation
  }
}
```

#### 2. New Reasoner Integration Service
```typescript
// New file: src/backend/services/mcp/reasoner-integration.ts
export interface FlowOptimizationResult {
  suggestions: OptimizationSuggestion[];
  alternativePaths: AlternativePath[];
  performanceMetrics: PerformanceMetrics;
  confidence: number;
  reasoning: string;
}

export interface OptimizationSuggestion {
  type: 'performance' | 'reliability' | 'efficiency' | 'user_experience';
  description: string;
  impact: 'low' | 'medium' | 'high';
  implementation: string;
  estimatedImprovement: number;
}

export interface AlternativePath {
  pathId: string;
  description: string;
  nodes: string[];
  estimatedPerformance: number;
  reasoning: string;
}

export class ReasonerIntegrationService {
  
  static async analyzeFlowContext(
    context: FlowContext
  ): Promise<ContextAnalysis> {
    return await MCPHandler.executeMCP({
      mcpServer: 'mcp-reasoner',
      enabledTools: ['mcp-reasoner'],
      context: {
        flowStructure: context.flow,
        executionHistory: context.history,
        userPreferences: context.preferences,
        systemConstraints: context.constraints
      },
      task: 'analyze_flow_context_and_provide_insights'
    });
  }
  
  static async suggestOptimizations(
    flow: ReactFlow,
    executionData: ExecutionData
  ): Promise<OptimizationSuggestion[]> {
    const result = await MCPHandler.executeMCP({
      mcpServer: 'mcp-reasoner',
      enabledTools: ['mcp-reasoner'],
      context: {
        flowDefinition: flow,
        performanceData: executionData.performance,
        errorHistory: executionData.errors,
        userFeedback: executionData.feedback
      },
      task: 'generate_flow_optimization_suggestions'
    });
    
    return result.value.suggestions || [];
  }
}
```

#### 3. Type Definitions
```typescript
// Add to existing types
export interface SharedState {
  // ... existing properties
  optimizationInsights?: FlowOptimizationResult;
  reasoningHistory?: ReasoningStep[];
}

export interface ReasoningStep {
  stepId: string;
  timestamp: number;
  reasoning: string;
  confidence: number;
  suggestions: OptimizationSuggestion[];
  appliedOptimizations: string[];
}
```

### Integration Points

1. **FlowExecutor.executeStep()**: Add pre-execution analysis
2. **MCPHandler**: Ensure mcp-reasoner server is properly configured
3. **SharedState**: Store reasoning insights for UI display
4. **Debug Dashboard**: Show reasoning insights (Task B3)

## Acceptance Criteria

### Functional Requirements
- [ ] FlowExecutor can call mcp-reasoner before flow execution
- [ ] Reasoning results are stored in SharedState
- [ ] Optimization suggestions are generated for flows
- [ ] Alternative execution paths are identified
- [ ] Confidence scores are provided for all suggestions

### Technical Requirements
- [ ] No breaking changes to existing FlowExecutor functionality
- [ ] Proper error handling for mcp-reasoner failures
- [ ] Performance impact < 100ms per reasoning call
- [ ] Integration tests pass for all reasoning scenarios

### UI Requirements (for follow-up tasks)
- [ ] Reasoning insights are available for UI display
- [ ] Optimization suggestions can be presented to users
- [ ] Alternative paths can be visualized in flow builder

## Dependencies

### Required Before Starting
- [ ] mcp-reasoner MCP server is installed and configured
- [ ] MCPHandler supports mcp-reasoner server type
- [ ] SharedState type definitions are updated

### Enables After Completion
- **Task B2**: Flow Analysis Service Implementation
- **Task B3**: Performance Optimization Suggestions UI
- **Task B4**: Sequential-Thinking Integration (Dynamic Branching)

## Testing Strategy

### Unit Tests
```typescript
describe('ReasonerIntegrationService', () => {
  it('should analyze flow context successfully', async () => {
    const context = createMockFlowContext();
    const analysis = await ReasonerIntegrationService.analyzeFlowContext(context);
    expect(analysis).toBeDefined();
    expect(analysis.insights).toBeArray();
  });
  
  it('should suggest optimizations for flows', async () => {
    const flow = createMockFlow();
    const executionData = createMockExecutionData();
    const suggestions = await ReasonerIntegrationService.suggestOptimizations(flow, executionData);
    expect(suggestions).toBeArray();
    expect(suggestions.length).toBeGreaterThan(0);
  });
});
```

### Integration Tests
```typescript
describe('FlowExecutor with Reasoning', () => {
  it('should execute flows with reasoning analysis', async () => {
    const sharedState = createMockSharedState();
    const result = await FlowExecutor.executeStep('step1', sharedState);
    expect(sharedState.optimizationInsights).toBeDefined();
    expect(result.success).toBe(true);
  });
  
  it('should handle mcp-reasoner failures gracefully', async () => {
    // Mock mcp-reasoner failure
    MockMCPHandler.simulateFailure('mcp-reasoner');
    const sharedState = createMockSharedState();
    const result = await FlowExecutor.executeStep('step1', sharedState);
    expect(result.success).toBe(true); // Should continue despite reasoning failure
  });
});
```

## Implementation Steps

### Phase 1: Core Integration (1 hour)
1. Update FlowExecutor.ts with reasoning analysis method
2. Create ReasonerIntegrationService.ts
3. Update type definitions for SharedState

### Phase 2: Testing & Validation (1 hour)  
1. Write unit tests for reasoner integration
2. Write integration tests for flow execution
3. Test error handling scenarios

### Phase 3: Documentation & Polish (30 minutes)
1. Update code documentation
2. Add JSDoc comments for all new methods
3. Update README if necessary

## Performance Considerations

- **Latency**: Reasoning analysis should add < 100ms to flow execution
- **Caching**: Consider caching reasoning results for identical flow contexts
- **Fallback**: Ensure flow execution continues even if reasoning fails
- **Resource Usage**: Monitor memory usage for complex reasoning operations

## Security Considerations

- **Input Validation**: Validate all inputs to mcp-reasoner
- **Error Handling**: Don't expose sensitive mcp-reasoner errors to users
- **Logging**: Log reasoning decisions for audit purposes
- **Access Control**: Ensure reasoning features respect user permissions

## Future Enhancements

This foundational implementation enables:

1. **Advanced Reasoning Strategies** (beam search, MCTS) - Task B2
2. **Real-time Optimization** during flow execution - Task B3
3. **User Feedback Integration** for learning - Task C5
4. **Community Pattern Sharing** - Task E5

## Related Documentation

- [GitHub Issue #5](https://github.com/joelfuller2016/FLUJO/issues/5) - Flow Intelligence Enhancement
- [MCP-Reasoner Documentation](../mcp-servers/code-reasoning/README.md)
- [FlowExecutor Architecture](../docs/architecture/flow-execution.md)
- [PROJECT_PLAN.md](../PROJECT_PLAN.md) - Category B Implementation

---

**Status**: Ready for Implementation  
**Next Task**: Task B2 - Flow Analysis Service Implementation  
**Updated**: June 3, 2025