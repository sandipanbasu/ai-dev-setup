# LLM Assisted Software Engineering

While most of the noise focuses on the code generation aspect, with good reasons, I don't recollect a good eval on code testing, code review, product prototyping - basically all other aspects for software engineering, which equally need to speed up, along with code generation. SWE Bench has a list, and these (e.g., deep code review) are classified as high-reasoning, long-running tasks with much higher costs.  

1. Part 1 - A Short Primer [Part 1: LLM Agents for Software Development — Fundamentals](./part-1-llm-agents-fundamentals.md)
    
2. Part 2 - Ruflo for OpenCode, Codex and Claude [RuFlo v3.5 — Universal MCP Integration Guide](./ruflo-mcp-integration-plan.md)
    
3. Part 3 - What does a daily life with an agent orchestrator look like, focusing on the task of code generation (touching reviewing and testing)
    
    [AI-Assisted Development Lifecycle with RuFlo](./ruflo-daily-workflow.md)
    
4. Part 4 - yet to be documented - How does an agent handle code review of MR 
5. Part 5 - yet to be documented - How to integrate P[laywright test agents](https://playwright.dev/docs/test-agents) to do automated testing 
6. Part 6 - yet to be documented - a Deploy agent, where while the mechanics of actual deploying is done well by CI/CD, the decision to deploy is still human-driven and I would really like to outsource that to an agent
7. Part 7 - yet to be documented - (PM Point of View) How to document and prototype a new feature using agents + Figma / raw HTML  
8. Part 8 - yet to be done and documented - Shortcut Manager (scrum master) - how to manage the shortcut tasks based on changing priorities, which in turn are documented in Slack and Notion pages. The agent’s objective here is to bring out the n most important thing that we  should do, tally with MR’s being raised and inform humans that there is alignment or deviation, etc 
9. Part 9 - yet to be done and documented - Production Logs Agents acting as L1+L2 support to look for bugs and inform the team - creates a shortcut ticket, hands to the build agent, which fixes, then to review age

Some notes - 

1. I want to emphasize that Ruflo is just one such orchestrator - and there are a lot of choices. Hence, whenever you read the term “ruflo” generalize that to an agent orchestrator 
2.