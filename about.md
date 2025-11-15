---
layout: default
title: About
---
```
class AboutMe:
    def __init__(self):
        self.name = "Deepak Swaminathan"
        self.status = "Actively looking for full-time roles" 
        self.current_role = "Freelance ML Specialist"
        self.location = "Newberry, FL"

    def get_current_focus(self):
        """What I am working on right now as a Freelancer & Researcher"""
        return {
            "Freelancing": "Refining LLMs & Statistical Analysis on Medical Data (EEG)",
            "Research": "Supervised Reinforcement Learning (SRL) with Unsloth",
            "Architecture": "Building Agentic AI workflows with LangGraph & vLLM"
        }

    def experience_highlights(self):
        """A quick look at my professional journey"""
        return [
            "Freelance: Applied Ridge/Lasso regression & evaluated LLM output quality",
            "Boston University: Research Assistant (RL in C++) & Teaching Assistant (Big Data)",
            "Sopra Steria: Deployed RAG pipelines (Llama 2) & IAM Security Models (XGBoost)"
        ]

    def featured_projects(self):
        """Scalable systems I have built"""
        return {
            "SRL Fine-Tuning": "Developed a Supervised Reinforcement Learning framework using Unsloth & Meta's Synthetic-Data-Kit",
            "Football Researcher": "Agentic AI (CoT/ReAct) for soccer analysis using LangGraph and vLLM",
            "Game of Tigers & Goats": "AlphaZero agent trained using JAX and Ray on multi-GPU",
            "Distributed Vision": "Object detection on 1M+ images using PySpark & GCP"
        }

    def contact_me(self):
        return {
            "Email": "23.deepak.s@gmail.com",
            "LinkedIn": "[linkedin.com/in/deepak-swaminathan](https://www.linkedin.com/in/deepak-swaminathan-90a707153/)",
            "Goal": "Ready to join a team tackling complex AI/ML challenges."
        }

if __name__ == "__main__":
    me = AboutMe()
    print(f"Is {me.name} available for hire? {me.status}")
    # Output: Is Deepak Swaminathan available for hire? Actively looking for full-time roles

```