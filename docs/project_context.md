# Explainable AI Framework for GraphRAG Systems via Multi-Agent Coordination

---

# Project Overview

This project is a MERN Stack based Explainable GraphRAG platform for academic literature discovery.

The primary goal is not just to answer research questions, but to transparently demonstrate HOW the answer was generated.

Unlike conventional RAG or GraphRAG systems that function as black boxes, this platform visualizes the reasoning process through multiple coordinated agents and an interactive knowledge graph.

---

# Problem Statement

Current research discovery platforms like Google Scholar, Semantic Scholar and Elicit retrieve relevant papers but do not provide transparent reasoning behind their recommendations.

Researchers cannot easily understand:

- Why a paper was selected
- Which citation paths were followed
- Which evidence was ignored
- How the final answer was synthesized

This reduces trust in AI-assisted literature discovery.

---

# Proposed Solution

Develop an Explainable Multi-Agent GraphRAG framework that:

• Retrieves academic metadata from OpenAlex and Semantic Scholar.

• Builds a temporary knowledge graph during query execution.

• Uses multiple specialized agents to solve complex research questions.

• Streams every reasoning step to the frontend.

• Visualizes graph traversal and evidence used for the final answer.

---

# Target Users

- Researchers
- Professors
- Postgraduate Students
- Undergraduate Students

---

# Technology Stack

Frontend
- React
- Vite
- Tailwind CSS
- Socket.io Client
- React Force Graph

Backend
- Node.js
- Express.js

Database
- MongoDB Atlas

External APIs
- OpenAlex API
- Semantic Scholar API

Communication
- Socket.io

Future AI Integration
- OpenAI / Ollama (To be decided later)

---

# High Level Architecture

User

↓

React Dashboard

↓

Express Backend

↓

Planner Agent

↓

Explorer Agent

↓

Critic Agent

↓

OpenAlex
Semantic Scholar
MongoDB Atlas

↓

Response

↓

React Dashboard

---

# Agent Responsibilities

Planner Agent

- Understand user intent
- Extract entities
- Create execution plan

Explorer Agent

- Retrieve papers
- Perform graph traversal
- Perform vector search (future)

Critic Agent

- Validate evidence
- Remove irrelevant information
- Ensure source consistency

---

# Core Features

- Natural language query
- Academic paper retrieval
- Explainability timeline
- Interactive graph visualization
- Multi-agent reasoning
- Evidence panel
- Paper details
- Live agent status

---

# UI Theme

- Professional
- Dark Theme
- Blue Accent
- Glassmorphism
- Modern Research Dashboard

---

# Development Strategy

Phase 1

Complete Frontend using dummy data.

Phase 2

Backend Integration

Phase 3

Knowledge Graph

Phase 4

Multi-Agent Coordination

Phase 5

Explainability

---

# Important Notes

This is an Explainable AI Framework.

GraphRAG is only one module.

Explainability is the primary contribution.

Always prioritize modular architecture and reusable components.