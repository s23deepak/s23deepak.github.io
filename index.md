---
layout: default
title: Home
---

# Welcome to My AI Journey!

I'm an **AI enthusiast**, deeply curious about the inner workings of artificial intelligence. I'm constantly exploring new techniques that tackle emerging challenges in the field.

In my journey so far, I've developed a Supervised Reinforcement Learning (SRL) fine-tuning framework using Unsloth, built custom data pipelines with Meta's Synthetic-Data-Kit and vLLM, and designed step-wise reward functions for dense RL. Currently, I'm contributing to the open-source Transformer Reinforcement Learning library—working to integrate SRL algorithms for broader adoption in the community.

My interests span machine learning pipelines, large language models (LLMs), reinforcement learning, MLOps, and open-source AI systems. I’m especially excited about agentic AI, model optimization, cloud training workflows, and collaborative community projects.

Here, I'll be sharing my learnings, insights, and discoveries as I dive deeper into the world of AI—including technical write-ups, project updates, and practical strategies.
Feel free to explore and join the conversation!
## Latest Posts

<ul class="post-list">
{% for post in site.posts %}
  <li>
    <a href="{{ post.url }}">{{ post.title }}</a>
    <span class="date">{{ post.date | date: "%B %d, %Y" }}</span>
  </li>
{% endfor %}
</ul>