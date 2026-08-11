# Main Architectures for AI Agents — Implementations and Code Snippets

Introduction

Designing AI agents forces you to pick a control philosophy: do you want rock‑solid reactivity, symbolic planning, learned policies, or a mixture? In practice engineers choose architectures that match latency, interpretability, and learning requirements. This post walks through the main agent architectures (reactive, deliberative, BDI, hybrid, reinforcement learning, and modern LLM-based agents), shows strengths/weaknesses, and gives concise runnable code snippets and tool links so you can prototype fast.

Overview of architectures

- Reactive (subsumption / behavior-based)
- Deliberative (STRIPS / PDDL planners)
- BDI (Belief–Desire–Intention)
- Hybrid (three-layer / planner + reactive arbitration)
- Reinforcement Learning (policy learning, single- and multi-agent)
- LLM-based agents (tool-using, ReAct-style)

1) Reactive architectures

High-level description

Reactive architectures connect perception to action with minimal symbolic state. Behavior modules run in parallel and are coordinated by priority, inhibition or suppression (Brooks' subsumption). These systems are simple, low-latency, and robust in noisy physical settings.

Strengths / Weaknesses (brief)

- Strengths: fast response, simple modules, robust to sensor noise.
- Weaknesses: poor long-term planning, hard to express complex goals.

Concrete implementation approach

Implement behaviors as small objects with a `consider(sensors)` method that returns whether they're active and an action. Use priorities to suppress lower-priority behaviors.

Runnable subsumption-style example (Python)

```python
# reactive_subsumption.py
import time

class Behavior:
    def __init__(self, name, priority):
        self.name = name
        self.priority = priority
    def consider(self, sensors):
        # return (active:bool, action:any, suppress:bool)
        raise NotImplementedError

class Wander(Behavior):
    def consider(self, sensors):
        return True, ('move_forward', 0.5), False

class AvoidObstacle(Behavior):
    def consider(self, sensors):
        if sensors.get('front_dist', 1.0) < 0.3:
            return True, ('turn', 0.8), True
        return False, None, False

def actuate(action):
    print('ACT:', action)

behaviors = [AvoidObstacle('avoid', 2), Wander('wander', 1)]
behaviors.sort(key=lambda b: b.priority, reverse=True)

sensors = {'front_dist': 1.0}
for t in range(20):
    sensors['front_dist'] = 0.2 if t in (5,6,7) else 1.0
    output = None
    for b in behaviors:
        active, action, suppress = b.consider(sensors)
        if active:
            if suppress:
                output = action
                break
            if output is None:
                output = action
    actuate(output)
    time.sleep(0.05)
```

Run this script with `python reactive_subsumption.py` to see the avoidance behavior suppress wandering when the simulated front distance is small.

Resources

- Subsumption architecture (Brooks): https://apps.dtic.mil/dtic/tr/fulltext/u2/a160833.pdf

2) Deliberative architectures (STRIPS / PDDL planners)

High-level description

Deliberative agents keep an explicit symbolic world model and compute plans (sequences of actions) using search/planning (STRIPS/PDDL). Planning is great when you can model the domain and need provable plans.

Strengths / Weaknesses

- Strengths: rich symbolic reasoning, verifiability, can produce optimal/short plans.
- Weaknesses: computational cost, less reactive under uncertainty.

Concrete implementation approach

You can either call an existing planner (pyperplan, Fast Downward) as a subprocess or embed a lightweight planner. For production robotics, planners are often top-layer and trigger plans that are executed by lower layers.

Toy STRIPS planner (Python BFS)

```python
# tiny_strips.py
from collections import deque

class Action:
    def __init__(self, name, pre, add, delete):
        self.name, self.pre, self.add, self.delete = name, set(pre), set(add), set(delete)

actions = [
    Action('pickup', ['at(box,table)'], ['holding(box)'], ['at(box,table)']),
    Action('putdown', ['holding(box)'], ['at(box,table)'], ['holding(box)'])
]

init = set(['at(robot,table)','at(box,table)'])
goal = set(['holding(box)'])

def applicable(a, s):
    return a.pre <= s

def apply(a, s):
    return (s - a.delete) | a.add

def bfs():
    q = deque([(init, [])])
    seen = {frozenset(init)}
    while q:
        s, plan = q.popleft()
        if goal <= s:
            return plan
        for a in actions:
            if applicable(a, s):
                ns = apply(a, s)
                fs = frozenset(ns)
                if fs not in seen:
                    seen.add(fs)
                    q.append((ns, plan+[a.name]))
    return None

if __name__ == '__main__':
    print('Plan:', bfs())
```

Calling external planners: pyperplan and Fast Downward

- pyperplan is a lightweight Python planner useful for learning and prototyping: https://github.com/aibasel/pyperplan

Example wrapper that runs pyperplan as a subprocess (requires pyperplan installed):

```python
# run_pyperplan.py
import subprocess
res = subprocess.run(['pyperplan','domain.pddl','problem.pddl'], capture_output=True, text=True)
print(res.stdout)
```

Fast Downward is a more capable planner; typical usage is to call the binary with a domain/problem PDDL. See: http://www.fast-downward.org/

Resources

- Pyperplan: https://github.com/aibasel/pyperplan
- Fast Downward: http://www.fast-downward.org/

3) BDI (Belief–Desire–Intention)

High-level description

BDI agents model mental states: Beliefs (information about the world), Desires (goals) and Intentions (committed plans). Systems like PRS, dMARS, Jason and 2APL implement this model and are popular for multi-agent systems.

Strengths / Weaknesses

- Strengths: natural mapping for goal-driven and multi-agent problems, good for commitment reasoning.
- Weaknesses: requires plan libraries and careful engineering; full logical models are computationally expensive.

Concrete implementation approach

Use an existing BDI platform (Jason for AgentSpeak, 2APL) or SPADE with BDI plugin for Python. Otherwise implement a small BDI loop: update beliefs from sensors, select/adopt goals, choose plans (or plan library entries), and execute intentions step-by-step.

Minimal BDI-like pseudocode (Python)

```python
# bdi_pseudocode.py
class BDI:
    def __init__(self):
        self.beliefs = set()
        self.goals = []
        self.intentions = []
    def add_belief(self,b): self.beliefs.add(b)
    def adopt_goal(self,g): self.goals.append(g)
    def deliberate(self):
        for g in list(self.goals):
            if g == 'deliver' and 'at(robot,locA)' in self.beliefs:
                self.intentions.append(['goto','locB','drop'])
                self.goals.remove(g)
    def execute(self):
        if self.intentions:
            step = self.intentions[0].pop(0)
            print('exec',step)

if __name__=='__main__':
    agent = BDI()
    agent.add_belief('at(robot,locA)')
    agent.adopt_goal('deliver')
    agent.deliberate(); agent.execute()
```

BDI platforms and links

- Jason (AgentSpeak interpreter, Java): http://jason.sourceforge.net/
- 2APL: https://2apl.sourceforge.net/
- SPADE & SPADE-BDI: https://spade.agentos.io/ and https://github.com/javipalanca/spade_bdi

4) Hybrid architectures (three-layer)

High-level description

Hybrid (three-layer) architectures combine a deliberative planner (top), an executive/sequencer (middle), and reactive behaviors (bottom). The executive arbitrates between executing plan steps and letting reactive controllers override when necessary.

Strengths / Weaknesses

- Strengths: combines planning with real-time reactivity; widely used in robotics.
- Weaknesses: arbitration and inter-layer consistency are tricky to design.

Concrete implementation approach

Top layer: PDDL planner or task planner. Middle: sequencer/state machine/behavior tree. Bottom: reactive controllers handling motor control and obstacle avoidance. Arbitration: executive can pause plan execution and request replanning when reactive layer intervenes.

Three-layer example (Python)

```python
# hybrid_three_layer.py
import time

# Planner stub
def plan(start, goal):
    return [('move','A->B'), ('move','B->C')]

class Sequencer:
    def __init__(self): self.queue=[]
    def load(self, plan): self.queue = list(plan)
    def next(self): return self.queue.pop(0) if self.queue else None

# Reactive layer
def reactive_execute(action, sensors):
    if sensors.get('obstacle'):
        return ('avoid',)
    return action

sensors = {'obstacle': False}
seq = Sequencer()
seq.load(plan('start','goal'))
for step in range(10):
    act = seq.next()
    if not act: break
    sensors['obstacle'] = (step==1)
    cmd = reactive_execute(act, sensors)
    print('Exec:', cmd)
    if cmd[0]=='avoid':
        print('Reactive handled obstacle; replan requested')
        seq.load(plan('now','goal'))
    time.sleep(0.1)
```

Frameworks & tools

- ROS for integration: http://ros.org
- Behavior trees: py_trees (Python) or BehaviorTree.CPP
- Planners: pyperplan, Fast Downward

5) Reinforcement Learning (RL)

High-level description

RL agents learn policies from interaction using reward signals (MDP/POMDP). Modern deep RL handles high-dimensional inputs and continuous control.

Strengths / Weaknesses

- Strengths: learns from data, can discover non-obvious policies.
- Weaknesses: sample-inefficient, sensitive to reward design, unstable training.

Implementation approach

Use libraries like Stable Baselines3 or RLlib. The agent loop in RL focuses on collect->learn->act (interleaving policy updates and environment interaction), which differs from planner-based systems where planning happens offline or at a higher level.

Tiny Gym example (policy loop differs from planner)

```python
# rl_gym_example.py
import gym
import numpy as np

env = gym.make('CartPole-v1')
obs = env.reset()
for t in range(100):
    action = env.action_space.sample()  # random policy (placeholder for learned policy)
    obs, reward, done, info = env.step(action)
    env.render()
    if done:
        obs = env.reset()
env.close()
```

RL libraries

- Stable Baselines3: https://stable-baselines3.readthedocs.io/
- RLlib (Ray): https://docs.ray.io/en/latest/rllib.html
- OpenAI Gym / Gymnasium: https://www.gymlibrary.dev/

6) LLM-based agents (ReAct / tool-using)

High-level description

Large language models can be used as controllers that generate reasoning traces and call external tools (search, calculators, web APIs). ReAct-style prompts interleave reasoning and action, which gives the model a way to plan and use tools.

Strengths / Weaknesses

- Strengths: strong commonsense, flexible natural-language plans, rapid prototyping for tool orchestration.
- Weaknesses: hallucination, brittleness, limited grounded acting without tool verification.

Concrete implementation approach

Use an LLM as a loop that: builds a prompt including context + tools, produces reasoning and tool calls, executes tools, and feeds results back until a final answer is produced. For reproducibility/mockability write wrappers that stub tool calls.

ReAct-style prompt pattern (short)

```
You are an agent. Use the following tools when needed: Search(query) -> text; Calculator(expr) -> number
Question: How many widgets will we need if price=...?
Thought: I should search for current widget price.
Action: Search("widget price 2026")
Observation: ...
Thought: I should calculate ...
Action: Calculator("12 * 3.5")
Observation: 42
Final Answer: ...
```

Minimal Python example using an OpenAI-like stubbed client

```python
# llm_agent_stub.py
import random

def fake_openai_call(prompt):
    # very small deterministic stub for demo
    if 'Search(' in prompt:
        return 'Observation: Found widget price 3.5 per unit'
    if 'Calculator(' in prompt:
        return 'Observation: 42'
    return 'Final Answer: Demo complete'

prompt = "You are an agent...\nAction: Search(\"widget price\")\n"
print(fake_openai_call(prompt))

# Example structure for real OpenAI client (commented)
# import openai
# def call_model(prompt):
#     return openai.ChatCompletion.create(model='gpt-4o-mini', messages=[{'role':'user','content':prompt}])
```

Tooling & patterns

- ReAct paper and pattern: https://arxiv.org/abs/2205.11916
- Agent frameworks: LangChain, LlamaIndex, AutoGPT families (use carefully)

Choosing an architecture — practical guidance

- Use reactive when latency and robustness matter (low-level robotics, embedded controllers).
- Use deliberative/STRIPS when you can model the domain and need correct, inspectable plans.
- Use BDI when you need clear goal/commitment semantics, multi-agent coordination, or a natural mapping to human intentions.
- Use hybrid three-layer as a pragmatic default in robotics: planner for missions, executive for sequencing, reactive for low-level control.
- Use RL when policies are hard to hand-code and you can provide simulation and reward signals.
- Use LLM agents for rapid prototyping of tool-using assistants, or when natural-language reasoning and broad knowledge are valuable — but add verification.

References and resources

- pyperplan (STRIPS planner): https://github.com/aibasel/pyperplan
- Fast Downward planner: http://www.fast-downward.org/
- SPADE BDI: https://github.com/javipalanca/spade_bdi
- Jason AgentSpeak: http://jason.sourceforge.net/
- Subsumption architecture (Brooks): https://apps.dtic.mil/dtic/tr/fulltext/u2/a160833.pdf
- The Landscape of Emerging AI Agent Architectures (survey): https://arxiv.org/html/2404.11584v1
- Stable Baselines3: https://stable-baselines3.readthedocs.io/
- RLlib (Ray): https://docs.ray.io/en/latest/rllib.html
- OpenAI API docs (example usage): https://platform.openai.com/docs

Conclusion

There is no one-size-fits-all architecture. Match your choice to constraints: latency, explainability, learning budget, and environment structure. Start small: prototype behaviors or a tiny planner, then move to hybridize once you need both reactivity and goal-directed behavior. If you want, I can expand any section into a full runnable repo with tests and ROS/Gazebo integration.

References

(Links collected above)
