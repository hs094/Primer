[[AI]] [[CrewAI]] 

**Crew AI** is an open-source framework designed to create multi-agent AI systems that collaborate to solve complex tasks. It allows developers to define roles, assign tasks, and coordinate multiple AI agents efficiently.

**1. Key Features of Crew AI**

✅ **Multi-Agent Collaboration** – Allows multiple AI agents to work together in a structured manner.

✅ **Role-Based Agents** – Each agent has a defined role, skills, and responsibilities.

✅ **Task Delegation** – Tasks can be assigned dynamically to different agents.

✅ **Memory & Context Retention** – Agents can store and recall information for better reasoning.

✅ **Easy Integration** – Works with LLMs (GPT-4, Claude, etc.) and external APIs.

**2. Basic Components of Crew AI**

1. **Agents** – AI-powered workers with specific roles and capabilities.

2. **Tasks** – Individual jobs assigned to agents.

3. **Crew** – A team of agents collaborating to complete a project.

4. **Tools** – External functions or APIs that agents can use.

5. **Memory** – Persistent storage to help agents retain information.

**3. How to Use Crew AI?**

**Step 1: Install Crew AI**

```
pip install crewai
```

**Step 2: Define Agents**

```
from crewai import Agent

researcher = Agent(
    role="Researcher",
    goal="Gather information on Crew AI",
    tools=["web_search"],
    memory=True
)
```

**Step 3: Create Tasks**

```
from crewai import Task

task1 = Task(
    description="Summarize Crew AI features",
    agent=researcher
)
```

**Step 4: Assemble the Crew**

```
from crewai import Crew

crew = Crew(agents=[researcher], tasks=[task1])
result = crew.kickoff()
print(result)
```

**4. Use Cases of Crew AI**

🔹 Automated Research Teams

🔹 Multi-Agent Chatbots

🔹 Task Automation & Workflow Management

🔹 Code Generation & Review

🔹 Content Curation & Summarization

**5. Why Use Crew AI?**

• **Efficient Multi-Agent Collaboration**

• **Structured Workflows**

• **Scalable & Modular**

• **Seamless LLM Integration**