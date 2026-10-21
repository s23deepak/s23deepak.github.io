---
layout: default
title: Home
---

<section class="hero-section">
    <h1 style="margin-top: 0; margin-bottom: 0.5rem;">AI Engineering &amp; Research</h1>
    <p class="hero-lead">
        Notes on reasoning models, agentic workflows, inference optimization, and open-source AI systems.
    </p>
    <p class="hero-bio">
        I am an ML engineer exploring reinforcement learning, scalable LLM inference, and agentic workflows. My current work spans Supervised Reinforcement Learning (SRL) with Unsloth, synthetic reasoning pipelines, and open-source contributions to Transformer Reinforcement Learning (TRL).
    </p>
</section>

<div class="section-heading">Selected Writings &amp; Technical Notes</div>

<div class="post-list">
{% for post in site.posts %}
  <a href="{{ post.url }}" class="post-card">
    <div class="post-meta-row">
      <span>{{ post.date | date: "%B %d, %Y" }}</span>
      <span>·</span>
      <span>{{ post.content | strip_html | number_of_words | divided_by: 200 | plus: 1 }} min read</span>
    </div>
    <h2 class="post-card-title">{{ post.title }}</h2>
    {% if post.categories and post.categories.size > 0 %}
    <div class="post-tag-list">
      {% for tag in post.categories %}
      <span class="post-tag">{{ tag }}</span>
      {% endfor %}
    </div>
    {% endif %}
  </a>
{% endfor %}
</div>
