# Task B4: Sequential-Thinking Integration for Dynamic Branching

**Task ID**: task-383  
**GitHub Issue**: #6 (Dynamic Branching Logic with Sequential Thinking)  
**Category**: B - Enhanced MCP Integration  
**Priority**: High (Depends on Task B1)  
**Estimated Time**: 4-5 hours  

## Overview

Replace FLUJO's static branching logic with intelligent, context-aware decision making using the `sequential-thinking` MCP server. This enables dynamic path selection in flows based on conversation context, execution history, and available options.

## Strategic Importance

Dynamic branching transforms FLUJO from a **predetermined workflow system** into an **adaptive AI orchestration platform**:

- **Context Awareness**: Decisions based on conversation history and current state
- **Intelligent Routing**: Optimal path selection using reasoning capabilities  
- **Adaptive Execution**: Flows that adapt to changing conditions
- **Chain of Thought**: Multi-step reasoning for complex decision making

## Technical Specifications

### Implementation Location
- **New Handler**: `src/backend/execution/flow/handlers/BranchingHandler.ts`
- **Enhanced Executor**: `src/backend/execution/flow/FlowExecutor.ts`
- **UI Components**: `src/frontend/components/Flow/BranchingVisualization/`
- **Dependencies**: sequential-thinking MCP server, Task B1 (mcp-reasoner)

### Core Implementation

#### 1. BranchingHandler Service
```typescript
// New file: src/backend/execution/flow/handlers/BranchingHandler.ts
export interface BranchingDecision {
  selectedNodes: string[];
  reasoning: string;
  confidence: number;
  alternativePaths: AlternativePath[];
  executionStrategy: 'sequential' | 'parallel' | 'conditional';
}

export interface BranchingContext {
  conversation: FlujoChatMessage[];
  availableOptions: MCPNodeConfig[];
  currentState: SharedState;
  executionHistory: ExecutionStep[];
  userPreferences?: UserPreferences;
  performanceConstraints?: PerformanceConstraints;
}

export class BranchingHandler {
  
  static async executeIntelligentBranching(
    context: BranchingContext
  ): Promise<BranchingDecision> {
    
    // Step 1: Analyze available options using sequential thinking
    const thinkingResult = await MCPHandler.executeMCP({
      mcpServer: 'sequential-thinking',
      enabledTools: ['sequentialthinking'],
      context: {
        conversation: context.conversation,
        availableNodes: context.availableOptions.map(node => ({
          id: node.id,
          type: node.type,
          capabilities: node.enabledTools,
          description: node.description,
          estimatedLatency: node.performanceMetrics?.averageLatency || 0,
          successRate: node.performanceMetrics?.successRate || 1.0
        })),
        currentState: {
          sharedVariables: context.currentState.sharedState,
          executionProgress: context.currentState.executionProgress,
          userContext: context.currentState.userContext
        },
        executionHistory: context.executionHistory.slice(-10), // Last 10 steps for context
        constraints: {
          maxLatency: context.performanceConstraints?.maxLatency || 5000,
          preferredStrategy: context.userPreferences?.executionStrategy || 'balanced',
          resourceLimits: context.performanceConstraints?.resourceLimits
        }
      },
      task: 'determine_optimal_branching_path_with_reasoning'
    });
    
    // Step 2: Validate reasoning with mcp-reasoner (if available)
    let validatedDecision = thinkingResult.value;
    
    try {
      const validation = await MCPHandler.executeMCP({
        mcpServer: 'mcp-reasoner',
        enabledTools: ['mcp-reasoner'],
        context: {
          proposedDecision: thinkingResult.value,
          originalContext: context,
          reasoningSteps: thinkingResult.value.reasoningSteps || []
        },
        task: 'validate_and_improve_branching_decision'
      });
      
      validatedDecision = validation.value.improvedDecision || thinkingResult.value;
    } catch (error) {
      // Fallback to sequential-thinking only if mcp-reasoner fails
      console.warn('mcp-reasoner validation failed, using sequential-thinking result:', error);
    }
    
    return {
      selectedNodes: validatedDecision.recommendedNodes || [],
      reasoning: validatedDecision.reasoning || 'No reasoning provided',
      confidence: validatedDecision.confidence || 0.5,
      alternativePaths: validatedDecision.alternatives || [],
      executionStrategy: validatedDecision.executionStrategy || 'sequential'
    };
  }
  
  static async evaluateBranchingOptions(
    options: MCPNodeConfig[],
    context: BranchingContext
  ): Promise<BranchingEvaluation[]> {
    
    const evaluations: BranchingEvaluation[] = [];
    
    for (const option of options) {
      const evaluation = await MCPHandler.executeMCP({
        mcpServer: 'sequential-thinking',
        enabledTools: ['sequentialthinking'],
        context: {
          nodeOption: option,
          conversationContext: context.conversation,
          currentState: context.currentState
        },
        task: 'evaluate_single_branching_option'
      });
      
      evaluations.push({
        nodeId: option.id,
        score: evaluation.value.score || 0,
        reasoning: evaluation.value.reasoning || '',
        estimatedOutcome: evaluation.value.outcome || '',
        risks: evaluation.value.risks || [],
        benefits: evaluation.value.benefits || []
      });
    }
    
    return evaluations.sort((a, b) => b.score - a.score);
  }
}
```

#### 2. Enhanced MCPNode Configuration
```typescript
// Update existing MCPNodeConfig interface
export interface MCPNodeConfig {
  // ... existing properties
  branchingStrategy: 'static' | 'intelligent' | 'hybrid';
  branchingCriteria?: {
    contextFactors: string[];           // What context to consider
    performanceThresholds: Record<string, number>;
    fallbackNodes: string[];            // Backup options
    decisionTimeout: number;            // Max time for decision
    confidenceThreshold: number;       // Minimum confidence required
  };
  performanceMetrics?: {
    averageLatency: number;
    successRate: number;
    resourceUsage: number;
  };
}
```

#### 3. FlowExecutor Integration
```typescript
// Modify existing FlowExecutor.ts
export class FlowExecutor {
  
  static async executeStep(
    stepId: string, 
    sharedState: SharedState
  ): Promise<ExecutionResult> {
    
    const step = sharedState.debugSteps.find(s => s.stepId === stepId);
    
    if (step?.type === 'mcp' && step.branchingStrategy === 'intelligent') {
      // NEW: Use intelligent branching for MCP nodes
      const branchingDecision = await this.executeIntelligentBranching(step, sharedState);
      
      // Update execution path based on intelligent decision
      if (branchingDecision.selectedNodes.length > 0) {
        return await this.executeBranchingDecision(branchingDecision, sharedState);
      }
    }
    
    // Continue with existing logic for static branching...
    return await this.executeStandardStep(step, sharedState);
  }
  
  private static async executeIntelligentBranching(
    currentStep: ExecutionStep,
    sharedState: SharedState
  ): Promise<BranchingDecision> {
    
    // Gather branching context
    const branchingContext: BranchingContext = {
      conversation: sharedState.messages || [],
      availableOptions: await this.getAvailableBranchingOptions(currentStep),
      currentState: sharedState,
      executionHistory: sharedState.debugSteps || [],
      userPreferences: sharedState.userPreferences,
      performanceConstraints: sharedState.performanceConstraints
    };
    
    return await BranchingHandler.executeIntelligentBranching(branchingContext);
  }
  
  private static async executeBranchingDecision(
    decision: BranchingDecision,
    sharedState: SharedState
  ): Promise<ExecutionResult> {
    
    // Store branching decision for debugging and learning
    sharedState.branchingHistory = sharedState.branchingHistory || [];
    sharedState.branchingHistory.push({
      timestamp: Date.now(),
      decision,
      actualOutcome: null  // Will be filled after execution
    });
    
    // Execute based on strategy
    switch (decision.executionStrategy) {
      case 'sequential':
        return await this.executeSequentialBranching(decision.selectedNodes, sharedState);
      case 'parallel':
        return await this.executeParallelBranching(decision.selectedNodes, sharedState);
      case 'conditional':
        return await this.executeConditionalBranching(decision, sharedState);
      default:
        return await this.executeSequentialBranching(decision.selectedNodes, sharedState);
    }
  }
}
```

### UI Components

#### 1. Branching Visualization Component
```typescript
// New file: src/frontend/components/Flow/BranchingVisualization/BranchingDebugPanel.tsx
export interface BranchingDebugProps {
  decision: BranchingDecision;
  context: BranchingContext;
  onApplyAlternative: (alternativePath: AlternativePath) => void;
}

export const BranchingDebugPanel: React.FC<BranchingDebugProps> = ({
  decision,
  context,
  onApplyAlternative
}) => {
  return (
    <div className="branching-debug-panel">
      <div className="decision-summary">
        <h3>Intelligent Branching Decision</h3>
        <div className="confidence-indicator">
          Confidence: {(decision.confidence * 100).toFixed(0)}%
        </div>
      </div>
      
      <div className="reasoning-display">
        <h4>Reasoning Process</h4>
        <div className="reasoning-text">{decision.reasoning}</div>
      </div>
      
      <div className="selected-paths">
        <h4>Selected Execution Path</h4>
        {decision.selectedNodes.map(nodeId => (
          <div key={nodeId} className="selected-node">
            {nodeId}
          </div>
        ))}
      </div>
      
      <div className="alternative-paths">
        <h4>Alternative Paths Considered</h4>
        {decision.alternativePaths.map((alt, index) => (
          <div key={index} className="alternative-path">
            <div className="alt-description">{alt.description}</div>
            <button 
              onClick={() => onApplyAlternative(alt)}
              className="apply-alternative-btn"
            >
              Use This Path
            </button>
          </div>
        ))}
      </div>
    </div>
  );
};
```

## Acceptance Criteria

### Functional Requirements
- [ ] Flows can make intelligent branching decisions based on context
- [ ] Sequential-thinking integration provides step-by-step reasoning
- [ ] Confidence scores are calculated for all branching decisions
- [ ] Alternative paths are identified and can be manually selected
- [ ] Fallback to static branching when intelligent branching fails

### Performance Requirements
- [ ] Branching decisions complete within 2 seconds
- [ ] System gracefully handles sequential-thinking server failures
- [ ] Memory usage remains stable during complex branching scenarios
- [ ] Parallel branching executes efficiently without resource conflicts

### UI Requirements
- [ ] Users can see reasoning behind branching decisions
- [ ] Alternative paths are visualized in debug panel
- [ ] Confidence indicators help users understand decision quality
- [ ] Manual override capability for branching decisions

## Dependencies

### Required Before Starting
- [ ] Task B1 (mcp-reasoner integration) completed
- [ ] sequential-thinking MCP server configured and tested
- [ ] MCPNodeConfig updated with branching strategy options
- [ ] SharedState enhanced with branching history storage

### Enables After Completion
- **Task B5**: Advanced Branching UI Components
- **Task B6**: Context-Aware Tool Selection
- **Task C1**: Memory integration for learning from branching decisions
- **Task D1**: Natural language flow generation with intelligent branching

## Testing Strategy

### Unit Tests
```typescript
describe('BranchingHandler', () => {
  describe('executeIntelligentBranching', () => {
    it('should make branching decisions based on context', async () => {
      const context = createMockBranchingContext();
      const decision = await BranchingHandler.executeIntelligentBranching(context);
      
      expect(decision.selectedNodes).toBeArray();
      expect(decision.reasoning).toBeString();
      expect(decision.confidence).toBeNumber();
      expect(decision.confidence).toBeGreaterThanOrEqual(0);
      expect(decision.confidence).toBeLessThanOrEqual(1);
    });
    
    it('should handle sequential-thinking server failures', async () => {
      MockMCPHandler.simulateFailure('sequential-thinking');
      const context = createMockBranchingContext();
      
      await expect(BranchingHandler.executeIntelligentBranching(context))
        .rejects.toThrow('Sequential thinking server unavailable');
    });
  });
  
  describe('evaluateBranchingOptions', () => {
    it('should score and rank branching options', async () => {
      const options = createMockNodeOptions();
      const context = createMockBranchingContext();
      const evaluations = await BranchingHandler.evaluateBranchingOptions(options, context);
      
      expect(evaluations).toBeArray();
      expect(evaluations).toBeSortedBy('score', { descending: true });
      evaluations.forEach(eval => {
        expect(eval.score).toBeNumber();
        expect(eval.reasoning).toBeString();
      });
    });
  });
});
```

### Integration Tests
```typescript
describe('FlowExecutor with Intelligent Branching', () => {
  it('should execute flows with intelligent branching decisions', async () => {
    const sharedState = createMockSharedState();
    const flow = createFlowWithIntelligentBranching();
    
    const result = await FlowExecutor.executeFlow(flow, sharedState);
    
    expect(result.success).toBe(true);
    expect(sharedState.branchingHistory).toBeArray();
    expect(sharedState.branchingHistory.length).toBeGreaterThan(0);
  });
  
  it('should fallback to static branching when intelligent branching fails', async () => {
    MockMCPHandler.simulateFailure('sequential-thinking');
    const sharedState = createMockSharedState();
    const flow = createFlowWithIntelligentBranching();
    
    const result = await FlowExecutor.executeFlow(flow, sharedState);
    
    expect(result.success).toBe(true); // Should still succeed with fallback
    expect(result.warnings).toContain('Intelligent branching failed, using static fallback');
  });
});
```

### User Acceptance Tests
```typescript
describe('Branching User Experience', () => {
  it('should provide clear reasoning for branching decisions', async () => {
    const { render, getByText } = renderBranchingDebugPanel();
    
    expect(getByText(/Reasoning Process/)).toBeInTheDocument();
    expect(getByText(/Confidence:/)).toBeInTheDocument();
    expect(getByText(/Alternative Paths/)).toBeInTheDocument();
  });
  
  it('should allow users to override branching decisions', async () => {
    const mockOnApplyAlternative = jest.fn();
    const { render, getByText } = renderBranchingDebugPanel({ 
      onApplyAlternative: mockOnApplyAlternative 
    });
    
    const applyButton = getByText('Use This Path');
    fireEvent.click(applyButton);
    
    expect(mockOnApplyAlternative).toHaveBeenCalled();
  });
});
```

## Implementation Steps

### Phase 1: Core Branching Logic (2-3 hours)
1. Create BranchingHandler.ts with intelligent branching methods
2. Update MCPNodeConfig interface with branching strategy options
3. Implement sequential-thinking integration for decision making

### Phase 2: FlowExecutor Integration (1-2 hours)
1. Modify FlowExecutor to support intelligent branching
2. Add branching decision storage to SharedState
3. Implement execution strategies (sequential, parallel, conditional)

### Phase 3: UI & Testing (1 hour)
1. Create BranchingDebugPanel component
2. Write comprehensive unit and integration tests
3. Add error handling and fallback mechanisms

## Performance Considerations

- **Decision Latency**: Aim for < 2 seconds per branching decision
- **Context Size**: Limit conversation history to last 10 messages for performance
- **Caching**: Cache branching decisions for identical contexts
- **Timeouts**: Implement timeouts for sequential-thinking calls
- **Resource Usage**: Monitor memory during parallel branching execution

## Security Considerations

- **Input Validation**: Validate all branching context data
- **Timeout Protection**: Prevent infinite loops in branching logic
- **Error Isolation**: Ensure branching failures don't crash flow execution
- **Audit Logging**: Log all branching decisions for security audits

## Future Enhancements

This implementation enables:

1. **Learning from Branching Outcomes** - Task C5 (Memory Integration)
2. **User Preference Learning** - Adapting decisions based on user feedback
3. **Community Pattern Sharing** - Sharing successful branching strategies
4. **Advanced Reasoning Strategies** - Integration with beam search and MCTS

## Related Documentation

- [GitHub Issue #6](https://github.com/joelfuller2016/FLUJO/issues/6) - Dynamic Branching Logic
- [Sequential-Thinking Documentation](../mcp-servers/servers/src/sequentialthinking/README.md)
- [Task B1 Documentation](./task-b1-mcp-reasoner-integration.md) - Prerequisite
- [FlowExecutor Architecture](../docs/architecture/flow-execution.md)

---

**Status**: Ready for Implementation (after Task B1)  
**Next Task**: Task B5 - Advanced Branching UI Components  
**Updated**: June 3, 2025