# 07. SOP Engine (Standard Operating Procedures)
## Enterprise Men's Clothing Manufacturing & Sales Platform

**Version:** 2.0  
**Last Updated:** March 26, 2026  
**Status:** ✅ Production-Ready

---

## 📋 Table of Contents

1. [SOP Engine Overview](#1-sop-engine-overview)
2. [Approval Workflow System](#2-approval-workflow-system)
3. [Purchase Order Approval](#3-purchase-order-approval)
4. [Discount Approval](#4-discount-approval)
5. [Return Authorization](#5-return-authorization)
6. [Quality Issue Escalation](#6-quality-issue-escalation)
7. [Agent Onboarding](#7-agent-onboarding)
8. [Workflow Builder](#8-workflow-builder)

---

## 1. SOP Engine Overview

### 1.1 Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                      SOP ENGINE ARCHITECTURE                      │
└──────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    Workflow Trigger Layer                        │
│  • API Endpoint Hooks                                            │
│  • Database Triggers                                              │
│  • Manual Initiation                                             │
│  • Scheduled Cron Jobs                                            │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Workflow Engine Core                          │
│  • Load workflow definition                                      │
│  • Initialize workflow instance                                  │
│  • Evaluate conditions                                           │
│  • Route to next step                                            │
│  • Track state transitions                                       │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Approval Step Processor                       │
│  • Assign to approver(s)                                         │
│  • Send notifications                                            │
│  • Wait for response                                             │
│  • Handle approval/rejection                                     │
│  • Apply escalation rules                                        │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Action Executor                               │
│  • Execute approved actions                                      │
│  • Update database records                                       │
│  • Trigger external APIs                                         │
│  • Send notifications                                            │
│  • Log completion                                                │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Database Schema

```sql
-- Workflow Definitions
CREATE TABLE sop_workflows (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    trigger_type VARCHAR(50) NOT NULL, -- manual, api, trigger, cron
    trigger_event VARCHAR(100),
    definition JSONB NOT NULL,
    is_active BOOLEAN DEFAULT true,
    version INTEGER DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Workflow Instances
CREATE TABLE sop_workflow_instances (
    id SERIAL PRIMARY KEY,
    workflow_id INTEGER REFERENCES sop_workflows(id),
    entity_type VARCHAR(100) NOT NULL, -- purchase_order, discount_request, etc.
    entity_id INTEGER NOT NULL,
    status VARCHAR(50) NOT NULL, -- pending, in_progress, completed, rejected, cancelled
    current_step VARCHAR(100),
    data JSONB,
    initiated_by INTEGER REFERENCES users(id),
    initiated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    completed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Approval Steps
CREATE TABLE sop_approval_steps (
    id SERIAL PRIMARY KEY,
    instance_id INTEGER REFERENCES sop_workflow_instances(id),
    step_name VARCHAR(255) NOT NULL,
    step_order INTEGER NOT NULL,
    approver_type VARCHAR(50) NOT NULL, -- user, role, conditional
    approver_id INTEGER REFERENCES users(id),
    approver_role VARCHAR(100),
    status VARCHAR(50) NOT NULL, -- pending, approved, rejected, skipped
    comments TEXT,
    responded_at TIMESTAMP,
    due_date TIMESTAMP,
    escalated BOOLEAN DEFAULT false,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Workflow Logs
CREATE TABLE sop_workflow_logs (
    id SERIAL PRIMARY KEY,
    instance_id INTEGER REFERENCES sop_workflow_instances(id),
    step_id INTEGER REFERENCES sop_approval_steps(id),
    action VARCHAR(100) NOT NULL,
    performed_by INTEGER REFERENCES users(id),
    details JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Escalation Rules
CREATE TABLE sop_escalation_rules (
    id SERIAL PRIMARY KEY,
    workflow_id INTEGER REFERENCES sop_workflows(id),
    step_name VARCHAR(255),
    timeout_hours INTEGER NOT NULL,
    escalate_to_role VARCHAR(100),
    escalate_to_user INTEGER REFERENCES users(id),
    notification_template TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_workflow_instances_status ON sop_workflow_instances(status);
CREATE INDEX idx_workflow_instances_entity ON sop_workflow_instances(entity_type, entity_id);
CREATE INDEX idx_approval_steps_approver ON sop_approval_steps(approver_id, status);
```

---

## 2. Approval Workflow System

### 2.1 Workflow Definition Format

```typescript
// Workflow Definition Structure
interface WorkflowDefinition {
  id: string;
  name: string;
  description: string;
  trigger: {
    type: 'manual' | 'api' | 'database_trigger' | 'cron';
    event?: string;
    conditions?: Condition[];
  };
  steps: WorkflowStep[];
  notifications: NotificationConfig[];
}

interface WorkflowStep {
  id: string;
  name: string;
  type: 'approval' | 'action' | 'condition' | 'parallel' | 'wait';
  config: any;
  onSuccess?: string; // Next step ID
  onFailure?: string; // Next step ID
  timeout?: number; // Hours
  escalation?: EscalationRule;
}

interface EscalationRule {
  afterHours: number;
  escalateTo: {
    type: 'role' | 'user';
    value: string | number;
  };
  notificationTemplate: string;
}
```

### 2.2 Workflow Engine Implementation

```typescript
// src/services/sop/workflow-engine.service.ts

import { prisma } from '../../lib/prisma';
import { WorkflowDefinition, WorkflowStep } from './types';

export class WorkflowEngine {
  /**
   * Start a new workflow instance
   */
  async startWorkflow(
    workflowId: number,
    entityType: string,
    entityId: number,
    initiatedBy: number,
    data: any = {}
  ) {
    // Load workflow definition
    const workflow = await prisma.sopWorkflows.findUnique({
      where: { id: workflowId },
    });

    if (!workflow || !workflow.isActive) {
      throw new Error('Workflow not found or inactive');
    }

    const definition: WorkflowDefinition = workflow.definition as any;

    // Create workflow instance
    const instance = await prisma.sopWorkflowInstances.create({
      data: {
        workflowId,
        entityType,
        entityId,
        status: 'in_progress',
        currentStep: definition.steps[0].id,
        data,
        initiatedBy,
      },
    });

    // Log workflow start
    await this.logAction(instance.id, null, 'workflow_started', initiatedBy, {
      workflowName: definition.name,
    });

    // Execute first step
    await this.executeStep(instance.id, definition.steps[0], definition);

    return instance;
  }

  /**
   * Execute a workflow step
   */
  async executeStep(
    instanceId: number,
    step: WorkflowStep,
    definition: WorkflowDefinition
  ) {
    const instance = await prisma.sopWorkflowInstances.findUnique({
      where: { id: instanceId },
    });

    if (!instance) {
      throw new Error('Workflow instance not found');
    }

    switch (step.type) {
      case 'approval':
        await this.executeApprovalStep(instanceId, step);
        break;

      case 'action':
        await this.executeActionStep(instanceId, step, definition);
        break;

      case 'condition':
        await this.executeConditionStep(instanceId, step, definition);
        break;

      case 'parallel':
        await this.executeParallelStep(instanceId, step, definition);
        break;

      default:
        throw new Error(`Unknown step type: ${step.type}`);
    }
  }

  /**
   * Execute approval step
   */
  async executeApprovalStep(instanceId: number, step: WorkflowStep) {
    const { approvers, approvalType } = step.config;

    // Create approval records
    for (let i = 0; i < approvers.length; i++) {
      const approver = approvers[i];

      await prisma.sopApprovalSteps.create({
        data: {
          instanceId,
          stepName: step.name,
          stepOrder: i + 1,
          approverType: approver.type,
          approverId: approver.type === 'user' ? approver.id : null,
          approverRole: approver.type === 'role' ? approver.role : null,
          status: 'pending',
          dueDate: step.timeout
            ? new Date(Date.now() + step.timeout * 3600000)
            : null,
        },
      });
    }

    // Send notifications to approvers
    await this.sendApprovalNotifications(instanceId, step, approvers);

    // Schedule escalation if timeout is set
    if (step.escalation) {
      await this.scheduleEscalation(instanceId, step);
    }
  }

  /**
   * Process approval response
   */
  async processApproval(
    stepId: number,
    approverId: number,
    action: 'approve' | 'reject',
    comments?: string
  ) {
    const step = await prisma.sopApprovalSteps.findUnique({
      where: { id: stepId },
      include: {
        instance: {
          include: {
            workflow: true,
          },
        },
      },
    });

    if (!step) {
      throw new Error('Approval step not found');
    }

    if (step.status !== 'pending') {
      throw new Error('Step already processed');
    }

    // Update step status
    await prisma.sopApprovalSteps.update({
      where: { id: stepId },
      data: {
        status: action === 'approve' ? 'approved' : 'rejected',
        respondedAt: new Date(),
        comments,
      },
    });

    // Log action
    await this.logAction(step.instanceId, stepId, action, approverId, {
      comments,
    });

    const instance = step.instance;
    const definition: WorkflowDefinition = instance.workflow.definition as any;
    const currentStepDef = definition.steps.find(
      (s) => s.id === instance.currentStep
    );

    if (!currentStepDef) {
      throw new Error('Current step definition not found');
    }

    // Check if all approvals are done
    const allSteps = await prisma.sopApprovalSteps.findMany({
      where: { instanceId: instance.id, stepName: step.stepName },
    });

    const approvalType = currentStepDef.config.approvalType;

    let shouldProceed = false;

    if (approvalType === 'all') {
      // All must approve
      shouldProceed =
        action === 'approve' &&
        allSteps.every((s) => s.status === 'approved' || s.id === stepId);
    } else if (approvalType === 'any') {
      // Any one approval is enough
      shouldProceed = action === 'approve';
    } else if (approvalType === 'majority') {
      // Majority must approve
      const approvedCount = allSteps.filter(
        (s) => s.status === 'approved'
      ).length;
      shouldProceed = approvedCount > allSteps.length / 2;
    }

    if (action === 'reject') {
      // Handle rejection
      await this.handleRejection(instance.id, currentStepDef, definition);
      return;
    }

    if (shouldProceed) {
      // Move to next step
      const nextStepId =
        action === 'approve'
          ? currentStepDef.onSuccess
          : currentStepDef.onFailure;

      if (nextStepId) {
        const nextStep = definition.steps.find((s) => s.id === nextStepId);
        if (nextStep) {
          await prisma.sopWorkflowInstances.update({
            where: { id: instance.id },
            data: { currentStep: nextStepId },
          });

          await this.executeStep(instance.id, nextStep, definition);
        }
      } else {
        // Workflow complete
        await this.completeWorkflow(instance.id, 'approved');
      }
    }
  }

  /**
   * Execute action step
   */
  async executeActionStep(
    instanceId: number,
    step: WorkflowStep,
    definition: WorkflowDefinition
  ) {
    const instance = await prisma.sopWorkflowInstances.findUnique({
      where: { id: instanceId },
    });

    if (!instance) {
      throw new Error('Workflow instance not found');
    }

    const { action, params } = step.config;

    try {
      // Execute the action
      switch (action) {
        case 'update_status':
          await this.executeUpdateStatus(instance, params);
          break;

        case 'send_email':
          await this.executeSendEmail(instance, params);
          break;

        case 'create_record':
          await this.executeCreateRecord(instance, params);
          break;

        case 'trigger_job':
          await this.executeTriggerJob(instance, params);
          break;

        default:
          throw new Error(`Unknown action: ${action}`);
      }

      // Log action completion
      await this.logAction(instanceId, null, 'action_executed', null, {
        action,
        params,
      });

      // Move to next step
      if (step.onSuccess) {
        const nextStep = definition.steps.find((s) => s.id === step.onSuccess);
        if (nextStep) {
          await prisma.sopWorkflowInstances.update({
            where: { id: instanceId },
            data: { currentStep: nextStep.id },
          });

          await this.executeStep(instanceId, nextStep, definition);
        }
      } else {
        // Workflow complete
        await this.completeWorkflow(instanceId, 'completed');
      }
    } catch (error) {
      // Handle failure
      await this.logAction(instanceId, null, 'action_failed', null, {
        error: error.message,
      });

      if (step.onFailure) {
        const failureStep = definition.steps.find(
          (s) => s.id === step.onFailure
        );
        if (failureStep) {
          await this.executeStep(instanceId, failureStep, definition);
        }
      } else {
        await this.completeWorkflow(instanceId, 'failed');
      }
    }
  }

  /**
   * Execute condition step
   */
  async executeConditionStep(
    instanceId: number,
    step: WorkflowStep,
    definition: WorkflowDefinition
  ) {
    const instance = await prisma.sopWorkflowInstances.findUnique({
      where: { id: instanceId },
    });

    const { condition } = step.config;

    // Evaluate condition
    const result = await this.evaluateCondition(condition, instance.data);

    const nextStepId = result ? step.onSuccess : step.onFailure;

    if (nextStepId) {
      const nextStep = definition.steps.find((s) => s.id === nextStepId);
      if (nextStep) {
        await prisma.sopWorkflowInstances.update({
          where: { id: instanceId },
          data: { currentStep: nextStepId },
        });

        await this.executeStep(instanceId, nextStep, definition);
      }
    } else {
      await this.completeWorkflow(instanceId, 'completed');
    }
  }

  /**
   * Complete workflow
   */
  async completeWorkflow(instanceId: number, status: string) {
    await prisma.sopWorkflowInstances.update({
      where: { id: instanceId },
      data: {
        status: status === 'approved' ? 'completed' : status,
        completedAt: new Date(),
      },
    });

    await this.logAction(instanceId, null, 'workflow_completed', null, {
      status,
    });
  }

  /**
   * Log workflow action
   */
  async logAction(
    instanceId: number,
    stepId: number | null,
    action: string,
    performedBy: number | null,
    details: any
  ) {
    await prisma.sopWorkflowLogs.create({
      data: {
        instanceId,
        stepId,
        action,
        performedBy,
        details,
      },
    });
  }

  // ... Additional helper methods
}
```

---

## 3. Purchase Order Approval

### 3.1 PO Approval Workflow

```
┌──────────────────────────────────────────────────────────────────┐
│                  PURCHASE ORDER APPROVAL FLOW                     │
└──────────────────────────────────────────────────────────────────┘

Purchase Order Created
     │
     ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 1: Amount-Based Routing                                    │
│                                                                   │
│  Amount < ₹50,000:                                               │
│  └──► Auto-approve ──► Send to Supplier                         │
│                                                                   │
│  Amount ₹50,000 - ₹2,00,000:                                    │
│  └──► Purchase Manager Approval                                  │
│                                                                   │
│  Amount > ₹2,00,000:                                             │
│  └──► Multi-level Approval                                       │
└────────┬────────────────────────────────────────────────────────┘
         │ (Amount > ₹50,000)
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 2: Purchase Manager Review                                 │
│  • Verify supplier credentials                                   │
│  • Check pricing reasonability                                   │
│  • Validate delivery timeline                                    │
│  • Review material specifications                                │
│                                                                   │
│  Timeout: 24 hours                                               │
│  Escalate to: Senior Purchase Manager                            │
└────────┬────────────────────────────────────────────────────────┘
         │
         ├──► Approved (Amount < ₹2,00,000) ──► Finance Verification
         │
         ▼ (Amount > ₹2,00,000)
┌─────────────────────────────────────────────────────────────────┐
│  Step 3: Finance Review                                          │
│  • Check budget availability                                     │
│  • Verify payment terms                                          │
│  • Review supplier payment history                               │
│  • Validate cost allocation                                      │
│                                                                   │
│  Timeout: 24 hours                                               │
│  Escalate to: Finance Manager                                    │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 4: Senior Management Approval (Amount > ₹2L)              │
│  • Strategic review                                              │
│  • Cash flow impact                                              │
│  • Supplier relationship                                         │
│  • Final authorization                                           │
│                                                                   │
│  Timeout: 48 hours                                               │
│  Escalate to: COO/CEO                                            │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 5: PO Execution                                            │
│  • Generate PO document                                          │
│  • Email to supplier                                             │
│  • Update system status                                          │
│  • Set delivery reminders                                        │
│  • Notify warehouse team                                         │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 PO Approval Definition

```json
{
  "id": "po_approval_v1",
  "name": "Purchase Order Approval Workflow",
  "description": "Multi-level approval for purchase orders based on amount",
  "trigger": {
    "type": "api",
    "event": "purchase_order.created"
  },
  "steps": [
    {
      "id": "amount_check",
      "name": "Check PO Amount",
      "type": "condition",
      "config": {
        "condition": {
          "field": "total",
          "operator": ">=",
          "value": 50000
        }
      },
      "onSuccess": "purchase_manager_approval",
      "onFailure": "auto_approve"
    },
    {
      "id": "purchase_manager_approval",
      "name": "Purchase Manager Approval",
      "type": "approval",
      "config": {
        "approvers": [
          {
            "type": "role",
            "role": "purchase_manager"
          }
        ],
        "approvalType": "any"
      },
      "timeout": 24,
      "escalation": {
        "afterHours": 24,
        "escalateTo": {
          "type": "role",
          "value": "senior_purchase_manager"
        },
        "notificationTemplate": "PO approval pending for 24 hours"
      },
      "onSuccess": "amount_check_finance",
      "onFailure": "reject_po"
    },
    {
      "id": "amount_check_finance",
      "name": "Check if Finance Review Needed",
      "type": "condition",
      "config": {
        "condition": {
          "field": "total",
          "operator": ">=",
          "value": 200000
        }
      },
      "onSuccess": "finance_approval",
      "onFailure": "approve_po"
    },
    {
      "id": "finance_approval",
      "name": "Finance Approval",
      "type": "approval",
      "config": {
        "approvers": [
          {
            "type": "role",
            "role": "finance_manager"
          }
        ],
        "approvalType": "any"
      },
      "timeout": 24,
      "onSuccess": "senior_management_approval",
      "onFailure": "reject_po"
    },
    {
      "id": "senior_management_approval",
      "name": "Senior Management Approval",
      "type": "approval",
      "config": {
        "approvers": [
          {
            "type": "role",
            "role": "senior_management"
          }
        ],
        "approvalType": "any"
      },
      "timeout": 48,
      "onSuccess": "approve_po",
      "onFailure": "reject_po"
    },
    {
      "id": "approve_po",
      "name": "Approve and Send PO",
      "type": "action",
      "config": {
        "action": "update_status",
        "params": {
          "status": "approved"
        }
      }
    },
    {
      "id": "reject_po",
      "name": "Reject PO",
      "type": "action",
      "config": {
        "action": "update_status",
        "params": {
          "status": "rejected"
        }
      }
    },
    {
      "id": "auto_approve",
      "name": "Auto Approve Small PO",
      "type": "action",
      "config": {
        "action": "update_status",
        "params": {
          "status": "approved"
        }
      }
    }
  ]
}
```

---

## 4. Discount Approval

### 4.1 Discount Approval Workflow

```
Discount Request Created
     │
     ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 1: Discount Type & Amount Check                           │
│                                                                   │
│  Standard Discount (<10%):                                       │
│  └──► Auto-approve                                               │
│                                                                   │
│  Medium Discount (10-20%):                                       │
│  └──► Sales Manager Approval                                     │
│                                                                   │
│  Large Discount (>20%):                                          │
│  └──► Multi-level Approval                                       │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 2: Sales Manager Review                                    │
│  • Verify customer eligibility                                   │
│  • Check order value                                             │
│  • Review customer history                                       │
│  • Validate business justification                               │
│                                                                   │
│  Timeout: 12 hours                                               │
└────────┬────────────────────────────────────────────────────────┘
         │
         ├──► Approved (Discount < 20%) ──► Apply Discount
         │
         ▼ (Discount > 20%)
┌─────────────────────────────────────────────────────────────────┐
│  Step 3: Finance Approval                                        │
│  • Calculate profit impact                                       │
│  • Review margin impact                                          │
│  • Check discount budget                                         │
│                                                                   │
│  Timeout: 24 hours                                               │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 4: Apply Discount                                          │
│  • Update order pricing                                          │
│  • Record discount usage                                         │
│  • Notify customer                                               │
│  • Update analytics                                              │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. Return Authorization

### 5.1 Return Approval Workflow

```
Return Request Created
     │
     ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 1: Automatic Checks                                        │
│  • Within return window (30 days)?                               │
│  • Product eligible for return?                                  │
│  • Order delivered & paid?                                       │
│  • Customer not blacklisted?                                     │
└────────┬────────────────────────────────────────────────────────┘
         │
         ├──► All Pass ──► Auto-approve (Standard Returns)
         │
         ▼ (Any Check Fails OR High Value)
┌─────────────────────────────────────────────────────────────────┐
│  Step 2: Customer Service Review                                 │
│  • Review return reason                                          │
│  • Check customer communication                                  │
│  • Verify product images                                         │
│  • Evaluate case priority                                        │
│                                                                   │
│  Timeout: 24 hours                                               │
└────────┬────────────────────────────────────────────────────────┘
         │
         ├──► Approved ──► Schedule Pickup
         │
         ▼ (Exception Case OR Value > ₹10,000)
┌─────────────────────────────────────────────────────────────────┐
│  Step 3: Manager Approval                                        │
│  • Review exception justification                                │
│  • Check return fraud patterns                                   │
│  • Evaluate customer lifetime value                              │
│  • Make final decision                                           │
│                                                                   │
│  Timeout: 24 hours                                               │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 4: Process Return                                          │
│  • Schedule pickup                                               │
│  • Generate return label                                         │
│  • Update order status                                           │
│  • Send confirmation email                                       │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6. Quality Issue Escalation

### 6.1 Quality SOP

```
Quality Issue Reported
     │
     ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 1: Severity Classification                                 │
│  • Minor: Cosmetic defect                                        │
│  • Major: Functional issue                                       │
│  • Critical: Safety concern                                      │
└────────┬────────────────────────────────────────────────────────┘
         │
         ├──► Minor ──► Quality Inspector ──► Approve Rework
         │
         ├──► Major ──► Quality Manager ──► Root Cause Analysis
         │
         └──► Critical ──► Immediate Escalation ──► Stop Production
                                │
                                ▼
                         Quality Head Review
                                │
                                ▼
                         Corrective Action Plan
                                │
                                ▼
                         Implementation & Verification
```

---

## 7. Agent Onboarding

### 7.1 Agent Onboarding SOP

```
┌──────────────────────────────────────────────────────────────────┐
│                  AGENT ONBOARDING WORKFLOW                        │
└──────────────────────────────────────────────────────────────────┘

New Agent Application
     │
     ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 1: Document Verification (Auto)                            │
│  • PAN Card                                                      │
│  • Aadhaar Card                                                  │
│  • Bank Account Details                                          │
│  • GST Number (if applicable)                                    │
│  • Address Proof                                                 │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 2: Background Check                                        │
│  • CIBIL score check                                             │
│  • Police verification (optional)                                │
│  • Reference verification                                        │
│  • Previous employment check                                     │
│                                                                   │
│  Timeout: 7 days                                                 │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 3: Sales Manager Interview                                 │
│  • Assess sales experience                                       │
│  • Territory knowledge                                           │
│  • Product knowledge test                                        │
│  • Commission structure discussion                               │
│                                                                   │
│  Timeout: 48 hours                                               │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 4: HR Approval                                             │
│  • Review application                                            │
│  • Verify all documents                                          │
│  • Approve compensation                                          │
│  • Generate agent ID                                             │
│                                                                   │
│  Timeout: 24 hours                                               │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 5: Onboarding & Training                                   │
│  • Create agent account                                          │
│  • Assign territory                                              │
│  • Provide training materials                                    │
│  • Schedule product training                                     │
│  • Issue sample kit                                              │
│  • Assign mentor                                                 │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 6: First Order Support                                     │
│  • Monitor first 10 orders                                       │
│  • Provide additional support                                    │
│  • Feedback collection                                           │
│  • Performance evaluation                                        │
│                                                                   │
│  Duration: 30 days probation                                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8. Workflow Builder

### 8.1 Visual Workflow Builder (UI Concept)

```typescript
// Workflow Builder Component Structure

interface WorkflowBuilderState {
  nodes: WorkflowNode[];
  connections: WorkflowConnection[];
  selectedNode: WorkflowNode | null;
}

interface WorkflowNode {
  id: string;
  type: 'start' | 'approval' | 'condition' | 'action' | 'end';
  position: { x: number; y: number };
  config: any;
}

interface WorkflowConnection {
  id: string;
  source: string;
  target: string;
  type: 'success' | 'failure' | 'default';
}

// Node Types Configuration
const nodeTypes = {
  approval: {
    icon: 'CheckCircle',
    label: 'Approval Step',
    configFields: [
      { name: 'approvers', type: 'multi-select', required: true },
      { name: 'approvalType', type: 'select', options: ['any', 'all', 'majority'] },
      { name: 'timeout', type: 'number', unit: 'hours' },
    ],
  },
  condition: {
    icon: 'GitBranch',
    label: 'Condition',
    configFields: [
      { name: 'field', type: 'text', required: true },
      { name: 'operator', type: 'select', options: ['==', '!=', '>', '<', '>=', '<='] },
      { name: 'value', type: 'text', required: true },
    ],
  },
  action: {
    icon: 'Zap',
    label: 'Action',
    configFields: [
      { name: 'actionType', type: 'select', required: true },
      { name: 'parameters', type: 'json' },
    ],
  },
};
```

### 8.2 Pre-built Workflow Templates

```yaml
Available Templates:
  1. Purchase Order Approval
  2. Discount Authorization
  3. Return Request Processing
  4. Agent Onboarding
  5. Quality Issue Escalation
  6. Customer Complaint Resolution
  7. Vendor Evaluation
  8. Budget Approval
  9. Product Launch Approval
  10. Marketing Campaign Approval

Each template includes:
  - Pre-configured steps
  - Default approvers (by role)
  - Timeout settings
  - Escalation rules
  - Notification templates
  - Customization options
```

---

**Document Version:** 2.0  
**Last Updated:** March 26, 2026  
**Next Review:** April 26, 2026  
**Status:** ✅ Production-Ready

---

[← Back to Index](./00_MASTER_INDEX.md) | [Next: Business Logic →](./08_BUSINESS_LOGIC.md)
