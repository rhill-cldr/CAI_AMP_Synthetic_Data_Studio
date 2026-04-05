# Synthetic Data Generation Application

> An application for generating high-quality synthetic datasets for various fine-tuning techniques. The application currently specializes in code generation and SQL query synthesis, with support for custom use cases.

## Table of Contents
- [Features](#features)
- [Architecture](#architecture)
- [Installation](#installation)
- [API Endpoints](#api-endpoints)
- [Usage Examples](#usage-examples)
- [Legal Notice](#legal-notice)

## Features

### Data Generation
- Support for multiple fine-tuning techniques:
  - ✅ Supervised Fine-Tuning (SFT) - Currently Live
  - 🚧 PPO (Proximal Policy Optimization) - WIP
  - 🚧 ORPO (Odds Ratio Preference Optimization) - WIP
  - 🚧 DPO (Direct Preference Optimization) - WIP
  - 🚧 KTO (Kahneman-Tversky Optimisation) - WIP

### Use Cases

#### Code Generation
Generate diverse programming question-answer pairs across multiple domains, complete with detailed explanations and working code examples. The system creates scenarios that test both theoretical understanding and practical implementation skills, producing high-quality training data for code-assistance models.

#### Text-to-SQL
Generate data as prompt and SQL pairs on custom data schemas which can be used to further fine-tune models for enhanced text2sql performance on OSS models.

#### Custom Use Cases
Flexible framework for implementing additional use cases which allows users to create their own workflow.

### Model Integration

#### AWS Bedrock Integration
- Claude 3 Family 
- Llama 3 Models
- Mistral Models

#### Cloudera AI Inference (CAII) Support


### Evaluation
- Built-in evaluation capabilities for generated datasets
- Customizable evaluation prompts
- Scoring system with detailed justifications

### Output Modes
- Preview Mode (displayed on Front-End) for prompts solution pairs <= 25
- Batch Mode via Cloudera ML Jobs (User can run this only in Cloudera environment)

## Architecture

Built using:
- Backend: FastAPI
- Frontend: React
- Database: SQLite (for metadata storage)

## Installation

1. Clone the repository:
   ```bash
   git clone https://github.com/cloudera/CAI_AMP_Synthetic_Data_Studio.git
   ```

2. Configure environment variables:
   ```bash
   # AWS Bedrock credentials (in CML environment)
   export AWS_ACCESS_KEY_ID="your key"
   export AWS_SECRET_ACCESS_KEY="your secret key"
   export AWS_DEFAULT_REGION="aws region"
   ```
   > Note: If using AWS Bedrock, ensure you have access to the LLM you intend to use.

3. Build the application:
   ```bash
   python build/build_client.py
   ```

4. Start the application:
   ```bash
   python build/start_application.py
   ```


##Aaccessing API endpoints directly.


1.  To generate/synthesise a sample for code generation you can use following code snippet
```python
import requests
import os
#********************Accessing Application**************************
# Get API key from environment variable if withinin CDSW app/session.
# To get your API key for using outside CDSW app/session follow given link.
# https://docs.cloudera.com/machine-learning/cloud/api/topics/ml-api-v2.html
api_key = os.environ.get('CDSW_APIV2_KEY')


# Below is your application API URL, you can look at swagger documentation 
# https://<application-subdomain>.<workbench-domain>/docs--> will take user to swagger documentaion
# Link to application can be found on application details page within CAI Workbench.


# URL for synthesis
url = https://<application-subdomain>.<workbench-domain>/synthesis/generate'

# Add the API key to headers with proper Authorization format
headers = {
    'accept': 'application/json',
    'Content-Type': 'application/json',
    'Authorization': f'Bearer {api_key}'  # Format as specified in the documentation
}   

# Pyload for data synthesis.
payload = {
    "use_case": "code_generation",
    "model_id": "us.anthropic.claude-3-5-haiku-20241022-v1:0",
    "num_questions": 3,
    "technique": "sft",
    "custom_prompt": "Requirements: - Each solution must include working code examples - Include explanations with the code - Follow the same format as the examples - Ensure code is properly formatted with appropriate indentation",
    "topics": ["python_basics", "data_structures"],
    "is_demo": True,
    "examples": [
        {
            "question": "How do you create a list in Python and add elements to it?",
            "solution": "Here's how to create and modify a list in Python:\n\n```python\n# Create an empty list\nmy_list = []\n\n# Add elements using append\nmy_list.append(1)\nmy_list.append(2)\n\n# Create a list with initial elements\nmy_list = [1, 2, 3]\n```"
        },
        {
            "question": "How do you implement a stack using a list in Python?",
            "solution": "Here's how to implement a basic stack:\n\n```python\nclass Stack:\n    def **init**(self):\n        self.items = []\n    \n    def push(self, item):\n        self.items.append(item)\n    \n    def pop(self):\n        if not self.is_empty():\n            return self.items.pop()\n    \n    def is_empty(self):\n        return len(self.items) == 0\n```"
        }
    ],
    "model_params": {
        "temperature": 0.0,
        "top_p": 1.0,
        "top_k": 250,
        "max_tokens": 4096
    }
}

# Add error handling to check if API key exists
if not api_key:
    print("Warning:No API key provided")


```

2. If you want to evaluate then you can se following code snippet. Make sure to use correct **import_path**.
```python

import requests
import os
#********************Accessing Application**************************
# Get API key from environment variable if withinin CDSW app/session.
# To get your API key for using outside CDSW app/session follow given link.
# https://docs.cloudera.com/machine-learning/cloud/api/topics/ml-api-v2.html
api_key = os.environ.get('CDSW_APIV2_KEY')


# Below is your application API URL, you can look at swagger documentation for all existing # endpoints for current application
# https://<application-subdomain>.<workbench-domain>/docs--> will take user to swagger documentaion
# Link to application can be found on application details page within CAI Workbench.


# URL for evaluation
url = https://<application-subdomain>.<workbench-domain>/synthesis/evaluate'

# Add the API key to headers with proper Authorization format
headers = {
    'accept': 'application/json',
    'Content-Type': 'application/json',
    'Authorization': f'Bearer {api_key}'  # Format as specified in the documentation
}   

# The prompt for evaluation
custom_prompt = """Below is a Python coding Question and Solution pair generated by an LLM. Evaluate its quality as a Senior Developer would, considering its suitability for professional use. Use the additive 5-point scoring system described below.

Points are accumulated based on the satisfaction of each criterion:
    1. Add 1 point if the code implements basic functionality and solves the core problem, even if it includes some minor issues or non-optimal approaches.
    2. Add another point if the implementation is generally correct but lacks refinement in style or fails to follow some best practices. It might use inconsistent naming conventions or have occasional inefficiencies.
    3. Award a third point if the code is appropriate for professional use and accurately implements the required functionality. It demonstrates good understanding of Python concepts and common patterns, though it may not be optimal. It resembles the work of a competent developer but may have room for improvement in efficiency or organization.
    4. Grant a fourth point if the code is highly efficient and follows Python best practices, exhibiting consistent style and appropriate documentation. It could be similar to the work of an experienced developer, offering robust error handling, proper type hints, and effective use of built-in features. The result is maintainable, well-structured, and valuable for production use.
    5. Bestow a fifth point if the code is outstanding, demonstrating mastery of Python and software engineering principles. It includes comprehensive error handling, efficient algorithms, proper testing considerations, and excellent documentation. The solution is scalable, performant, and shows attention to edge cases and security considerations."""

# Examples for evaluation
examples = [
    {
        "score": 3,
        "justification": """The code achieves 3 points by implementing core functionality correctly (1), 
        showing generally correct implementation with proper syntax (2), 
        and being suitable for professional use with good Python patterns and accurate functionality (3). 
        While it demonstrates competent development practices, it lacks the robust error handling 
        and type hints needed for point 4, and could benefit from better efficiency optimization and code organization."""
    },
    {
        "score": 4,
        "justification": """
        The code earns 4 points by implementing basic functionality (1), showing correct implementation (2), 
        being production-ready (3), and demonstrating high efficiency with Python best practices 
        including proper error handling, type hints, and clear documentation (4). 
        It exhibits experienced developer qualities with well-structured code and maintainable design, though 
        it lacks the comprehensive testing and security considerations needed for a perfect score."""
    }
]

# Model parameters
model_params = {
    "temperature": 0.0,
    "top_p": 1.0,
    "top_k": 250,
    "max_tokens": 4096
}

payload = {
    "export_type": "local",
    "display_name": "eval_code_gen",
    "import_path": "qa_pairs_llama_20241211_172456_final.json",
    "import_type": "local",
    "model_id": "us.anthropic.claude-3-5-sonnet-20241022-v2:0",
    "use_case": "code_generation",
    "is_demo": True,
    "custom_prompt": custom_prompt,
    "examples": examples,
    "model_params": model_params
}

response = requests.post(url, headers=headers, json=payload)

# Print the response
print(response.status_code)
print(response.json())

```

 

## Developer's Guide

The full developer documentation is published as an mdbook at `docs/current/`. To build locally:

```bash
cd docs/current && mdbook build
```

The guide covers architecture, API specification, data models, pipeline internals, export destinations, and integration patterns. See [docs/current/src/SUMMARY.md](docs/current/src/SUMMARY.md) for the table of contents.

## [Technical Overview](docs/technical_overview.md):

The given document gives overall capabilities, tech stack, and general idea to get started on this application.

## Technical Guides:
### [Generation Workflow](docs/guides/sft_workflow.md)
### [Evaluation Workflow](docs/guides/evaluation_workflow.md)



## Legal Notice

**IMPORTANT**: Please read the following before proceeding. This AMP includes or otherwise depends on certain third party software packages. Information about such third party software packages are made available in the notice file associated with this AMP. By configuring and launching this AMP, you will cause such third party software packages to be downloaded and installed into your environment, in some instances, from third parties' websites. For each third party software package, please see the notice file and the applicable websites for more information, including the applicable license terms.

If you do not wish to download and install the third party software packages, do not configure, launch or otherwise use this AMP. By configuring, launching or otherwise using the AMP, you acknowledge the foregoing statement and agree that Cloudera is not responsible or liable in any way for the third party software packages.

Copyright (c) 2024 - Cloudera, Inc. All rights reserved.