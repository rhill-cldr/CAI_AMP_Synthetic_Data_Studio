# Synthetic Data Studio: Technical Overview

## Introduction

Synthetic Data Studio is a comprehensive application designed to generate high-quality synthetic datasets, specifically tailored for large language model (LLM) fine-tuning and alignment. It employs a dual-architecture approach, integrating a Python-based backend with a React frontend while leveraging Cloudera AI enterprise services.

## Core Capabilities

### Supervised Fine-Tuning Dataset Generation

The application specializes in generating datasets for supervised fine-tuning of machine learning models. This functionality allows organizations to create task-specific training data while ensuring control over data quality and distribution.

### Document-Based Generation

Synthetic Data Studio incorporates advanced document processing capabilities, enabling users to generate synthetic data from uploaded document collections. This feature ensures the creation of domain-specific datasets that maintain consistency with existing documentation and knowledge bases.

### Custom Generation Workflows

The application implements a flexible workflow system that processes user-provided inputs to create customized data generation pipelines. This approach ensures that generated data aligns precisely with specific use cases and requirements.

### Model Alignment Data Generation

Supporting advanced reinforcement learning algorithms such as Direct Preference Optimization (DPO) and KTO, the system generates high-quality alignment data. This capability is essential for developing models that exhibit desired behaviors and adhere to specific performance criteria.

### Evaluation Workflow

The application also supports the evaluation of generated SFT datasets using LLM as a judge. It provides scores and corresponding justifications for generated prompt-completion pairs. Users can define their own scoring criteria and justification parameters, enabling more precise quality assessments. This feature helps filter and refine datasets to maintain high-quality standards before using them for supervised fine-tuning.

### Export

The generated and evaluated datasets can be exported to the Project File System. Additionally, datasets can be seamlessly uploaded to Hugging Face for broader accessibility and further model training.

## Technical Architecture

### Backend Implementation

The backend is developed in Python, providing robust data processing capabilities and efficient integration with machine learning workflows. This ensures seamless interaction with widely-used data science libraries and frameworks.

### Frontend Design

The frontend is built using React, offering a responsive and intuitive user interface. This modern web framework ensures efficient state management and smooth user interactions throughout the data generation process.

### Enterprise Integration

Synthetic Data Studio integrates with two major enterprise inference services:

- **AWS Bedrock**: Provides scalable cloud-based inference
- **Cloudera AI Inference Service**: Offers enterprise-grade deployment options

---

# Synthetic Data Studio: Prompt Engineering Guide

## Overview

This guide outlines the prompt engineering system employed in Synthetic Data Studio for generating high-quality synthetic data. The system utilizes a sophisticated template-based approach that incorporates examples, requirements, and dynamic seed injection.

## Prompt Template Structure

### Base Template

The core prompt template follows a structured format with three main components:

1. **Examples**: Demonstration cases for reference
2. **Requirements**: Specification of output format and quality
3. **Topic/Seed Injection**: Dynamic seed insertion for generating diverse outputs

### Example Prompt Template

```python
prompt_template = """
<examples>
{examples}
</examples>
{custom_prompt}
<topic>{seed}</topic>"""
```
#### Custom Prompt
```json
{
"""Write a programming question-pair for the following topic:

Requirements:
- Each solution must include working code examples
- Include explanations with the code
- Follow the same format as the examples
- Ensure code is properly formatted with appropriate indentation"""
}
```

#### Example Dataset

```json
{
   "question": "How do you read a CSV file into a pandas DataFrame?",
   "solution": """You can use pandas.read_csv(). Here's an example
   
        import pandas as pd
        df = pd.read_csv('data.csv')
        print(df.head())
        print(df.info())
"""
}
```

#### Seed Examples

```json
[
    "Implementation of QuickSort algorithm",
    "Depth-First Search traversal in graphs",
    "Dynamic programming solution for knapsack problem",
    "Implementation of A* pathfinding algorithm",
    "Binary search implementation with complexity analysis"
]
```

### Complete Prompt Example

```python
<examples>
[{
   "question": "How do you implement a graph data structure and perform depth-first search traversal?",
   "solution": """Here's an implementation of a graph using an adjacency list and depth-first search traversal:
   
class Graph:
    def __init__(self):
        self.graph = {}
    
    def add_vertex(self, vertex):
        if vertex not in self.graph:
            self.graph[vertex] = []
    
    def add_edge(self, vertex1, vertex2):
        self.add_vertex(vertex1)
        self.add_vertex(vertex2)
        self.graph[vertex1].append(vertex2)
        self.graph[vertex2].append(vertex1)
    
    def dfs(self, start_vertex):
        visited = set()
        
        def dfs_recursive(vertex):
            visited.add(vertex)
            print(f'Visiting vertex: {vertex}')
            for adjacent in self.graph[vertex]:
                if adjacent not in visited:
                    dfs_recursive(adjacent)
        
        dfs_recursive(start_vertex)

# Example usage
graph = Graph()

edges = [(1,2), (1,3), (2,4), (2,5), (3,4)]
for v1, v2 in edges:
    graph.add_edge(v1, v2)

graph.dfs(1)
"""},
{'question': 'How do you read a CSV file into a pandas DataFrame?', 
'solution': """You can use pandas.read_csv(). Here's an example:
import pandas as pd
df = pd.read_csv('data.csv')\n
"""}]
</examples>

Write a programming question-answer pair for the following topic:
Requirements:
- Each solution must include working code examples
- Include explanations with the code
- Follow the same format as the examples
- Ensure code is properly formatted with appropriate indentation
<topic>Depth-First Search traversal in graphs</topic>
```

---

# Synthetic Data Studio: Metadata Management

## Overview

The Synthetic Data Studio maintains metadata for each generation, evaluation, and export request using a SQLite database (`metadata.db`). This ensures traceability, reproducibility, and better dataset management.

## Metadata Tables

### **Generation Metadata (`generation_metadata.db`)**
Tracks metadata for synthetic data generation requests, including:
- Model ID
- Inference type
- Model parameters
- Prompt templates
- Seed values
- Example datasets

### **Evaluation Metadata (`evaluation_metadata.db`)**
Stores details of dataset evaluations, including:
- Model scores
- Justification criteria
- Prompt-completion pairs
- User-defined evaluation parameters

### **Export Metadata (`export_metadata.db`)**
Maintains logs of dataset exports, capturing:
- Destination (Project File System, Hugging Face, etc.)
- Export formats
- Associated metadata for version control

```python
import sqlite3
import pandas as pd
conn = sqlite3.connect("metadata.db")
query = "SELECT * FROM generation_metadata"
df = pd.read_sql_query(query, conn)
conn.close()
df.head()
```

---

## Full Developer's Guide

For comprehensive documentation covering API specification, pipeline internals, data models, and integration patterns, see the [Developer's Guide](current/src/SUMMARY.md) in `docs/current/`.