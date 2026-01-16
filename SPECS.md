# Malamar Project

Malamar is a personal project of Irving Dinh (me), it is designed to solve my problems:
1. I've Claude Code, Gemini CLI and also Codex CLI.
2. I want them working with each other, to checking other works, back to back. When everything is done, it will move to the in review and wait for my touching.

## Entities

### Workspace

Workspace is the roof of everything in Malamar. 
1. Workspace has the title (for me) and the description (for the agents) to understand the overall context of the tasks inside the workspace.
2. Workspace has multiple agents, usually: Planner; Implementer; Reviewer; Approver.
    2.1. Planner is the AI agent to ensure the requirement of the task is already clear enough, by asking me (as comments) or trying to do that by itself by using all the tools provided.
    2.2. Implementer is the AI agent to acutally working on the task, maybe a writing document task, or maybe a coding task, endless of possibility.
    2.3. Reviewer is the AI agent to double check the work of the implementer.
    2.4. Approver checking if the work is good enough, anything else to clarify before requesting for my attention.
3. Workspace has multiple tasks, each task has the summary (title) and the description. I will be the one who create the task, then the AI agents will working with each other fully autonomous to complete the task.

### Agents

Each agent has the instruction for example:
1. Planner: You're researcher, your responsibility to try your best to enriching the task's summary and requirement as much as possible, use all the tools you're provided, to make a plan to implement the task. If you found that the requirement is dangerously not clear enough, comment your questions and move the ticket to "in review" to draw my attention. Your plan should be in detailed and can be verified, comment your plan into the task so the implementer and the reviewer can working and verifying them.
2. Implementer: You're implementer, your responsibility to implement the task based on its description and the plan of the planner, if there is no plan, please do nothing. You're also responsible to verify the feedback of the reviewer, push back if needed to get the agreement on the improvements/fixes via the comments, then implement them if any.
3. Reviewer: You're reviewer, your responsibility to review the task based on its description, the plan of the planner and the working result of the implementer. You will try your best to ensure the working result meet the industrial quality, comment into the task and discuess with the implementer until the works are ready to be shipped.
4. Approver: You're approver, your responsibility is wait until everyone has on the same page and agree that the task is done, you will try your best to verify the result against the task, the plan and the feedback loop, then comment into the task to seeking for clarification if needed. When you see the result is good enough to be shipped, move the ticket to "in review" so the human can be looped into the task to review the result.

Agent also be setup to be assigned into an AI CLI (if many is available), for example:
1. Planner is assigned to Claude Code
2. Implementer is assigned to Codex CLI
3. Reviewr is assigned to Gemini CLI
4. Approver is assigned to Claude Code

The agent is all about the system instruction, where it will pasted into the Claude Code/Gemini CLI/Codex CLI/etc. when running.

### Tasks

Each task has the status, of:
1. Todo: Default, when I just created the task.
2. In progress: The agents are picked up and actively working on it.
3. In review: When the agents are agree with each other nothing to do with the task, it will be moved to in review, waiting for my instruction.
4. Done: I move the task here just for historical data

Most important, the task has the comments, that is the channel for all task's stakeholder to work with the tasks. The comments can comming from:
1. The user
2. The agents
3. The system, for note if there is an error.

### Task Router/Runner

Task router take the responsibilty to bind the task with the agent. When there is a task event has been queued and picked up, the task router will transfer the status:
1. If the task is in "Todo", move it to "In Progress"
2. If the task is in "In Porogress", but there is no action from the agents in the loop, move it "In Review" automatically.
3. If the task is in "In Review", but there is a comment of the user, move it to "In Progress" back automatically, and retrigger the loop.
4. If the task is in "Done", do nothing.

The task event is everything:
1. The user create a new task
2. The user/agent/system comment into a task
3. The task status/properties has been changed (title/description/etc.)

There is a queue for the task events to be put into the loop:
1. Each task has a event queue, to be picked into the task loop
2. Whenever a new task event has been emitted, if the queue of the task in empty, a new item will be added, the status will be queued
2. Whenever the loop is picking up the queue item to working on in the loop, the status will be changed to "in_progress"
3. If there is a new task event has been emitted while there is a queue item of that task is in "in_progress", a new item will be added, the status will be queued
4. If there is a new task even has been emitted while there is a queue item of that task is in "in_progress", and also another item of that task is in "queued", there is NO queue item of the task to be added.

Now we go to the loop, this is the core concept that making the Malamar:
1. The task router/runner will continously pickup the queue item, but limited into workspace, each workspace only has one queue item is being processed at a time.
2. After picked up the queue item, the runner will collect all of the data of the task, also of the workspace, then route it to each agents sequencially.
3. With each agent, Malamar will trigger the assigend CLI (e.g. Claude Code), paste in all the task, task comments, workspace intruction, to let the CLI to working on the task. The working directory will be a newly created (if not existed) in the temporary folder of the OS (/tmp/malamar_tasks_{task_id}) - OR, a static working directory setup in the workspace setting.
4. The CLI worked on the task, response to the runner on its actions: skip (has nothing to do), comment, change status. The runner does those actions. Then move to another agents in the list. As the agents are working in the same directory (temp, or static), they can indirectly share the working resource with each other, and summarize the note into the comment.
5. Redo the loop.

### Tasks Workflow

This is the use case scenario:
1. I create a new task with a detailed description. (A new task event queued)
2. Task router picking up the task

### Task event queue

## Technical Decision Note

### Miscellaneous

1. All the ID should be in nanoid format.

## Reference: 
