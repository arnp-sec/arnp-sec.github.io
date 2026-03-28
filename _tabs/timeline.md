---
layout: page
title: "Learning Timeline"
icon: fas fa-book
permalink: /timeline/
description: "My learning journey — certifications, modules, publications and milestones."
---

<link rel="stylesheet" href="{{ '/assets/css/timeline.css' | relative_url }}">

A living record of the journey. Not to show off, but to remember where each stone was placed. Every certification, experiment, and article represents a decision to go deeper rather than wider.

This timeline is primarily a personal historical record of my learning path and the road taken. Each point represents a particularly formative or meaninful moment in my progression.

The list is intentionally non-exhaustive. Rather than cataloguing every completed HackTheBox machine or Academy module, only the most significant milestones are documented here. The goal is signal over noise and to be able to clearly log each key topic from this learning journey.

Finally, no points related to professional experience are listed here, for confidentiality reasons. Findings, vulnerability discoveries, and security assessments conducted in professional contexts remain undocumented on this page by design.

<div class="timeline-legend" 
     style="margin-bottom: 2rem; display: flex; gap: 1.5rem; flex-wrap: wrap; font-size: 0.9rem;">
  
  <span class="legend-item active" data-category="certification"
        style="display:flex; align-items:center; gap:6px; cursor:pointer; 
               transition: opacity 0.2s;">
    <span style="display:inline-block; width:12px; height:12px; border-radius:50%; 
                 background:#f0a500;"></span>
    <span style="color:#f0a500; font-weight:500;">Certification</span>
  </span>

  <span class="legend-item active" data-category="course"
        style="display:flex; align-items:center; gap:6px; cursor:pointer;
               transition: opacity 0.2s;">
    <span style="display:inline-block; width:12px; height:12px; border-radius:50%; 
                 background:#00d4aa;"></span>
    <span style="color:#00d4aa; font-weight:500;">Course / Training</span>
  </span>

  <span class="legend-item active" data-category="htb_machine"
        style="display:flex; align-items:center; gap:6px; cursor:pointer;
               transition: opacity 0.2s;">
    <span style="display:inline-block; width:12px; height:12px; border-radius:50%; 
                 background:#9fef00;"></span>
    <span style="color:#9fef00; font-weight:500;">HTB Machine</span>
  </span>

  <span class="legend-item active" data-category="blog"
        style="display:flex; align-items:center; gap:6px; cursor:pointer;
               transition: opacity 0.2s;">
    <span style="display:inline-block; width:12px; height:12px; border-radius:50%; 
                 background:#007bff;"></span>
    <span style="color:#007bff; font-weight:500;">Article</span>
  </span>

  <span class="legend-item active" data-category="book"
        style="display:flex; align-items:center; gap:6px; cursor:pointer;
               transition: opacity 0.2s;">
    <span style="display:inline-block; width:12px; height:12px; border-radius:50%; 
                 background:#6f42c1;"></span>
    <span style="color:#6f42c1; font-weight:500;">Book</span>
  </span>

  <span class="legend-item active" data-category="lab"
        style="display:flex; align-items:center; gap:6px; cursor:pointer;
               transition: opacity 0.2s;">
    <span style="display:inline-block; width:12px; height:12px; border-radius:50%; 
                 background:#e83e8c;"></span>
    <span style="color:#e83e8c; font-weight:500;">Experiment</span>
  </span>

</div>

<div class="timeline">
  {% assign achievements = site.data.achievements %}
  {% for item in achievements %}
  
  <div class="timeline-item" data-category="{{ item.category }}">
    <div class="timeline-dot {{ item.category }}"></div>
    
    <div class="timeline-content">
      
      <div class="timeline-date">{{ item.date | date: "%B %d, %Y" }}</div>
      
      <div class="timeline-title">
        {% if item.link %}
          <a href="{{ item.link }}" target="_blank" rel="noopener noreferrer">
            {{ item.title }}
          </a>
        {% else %}
          {{ item.title }}
        {% endif %}
      </div>
      
      {% if item.description %}
        <p class="timeline-description">{{ item.description }}</p>
      {% endif %}

      {% if item.status == "in_progress" %}
        <span class="timeline-badge timeline-badge--in-progress">
          In progress{% if item.progress %} · {{ item.progress }}{% endif %}
        </span>
      
      {% elsif item.status == "paused" %}
        <span class="timeline-badge timeline-badge--paused">
          On hold{% if item.progress %} · {{ item.progress }}{% endif %}
        </span>
      
      {% endif %}
      
      {% if item.certificate %}
        <a href="{{ item.certificate }}" 
           target="_blank" 
           rel="noopener noreferrer" 
           class="timeline-certificate-btn">
          View Certificate
        </a>
      {% endif %}
      
    </div>
  </div>
  
  {% endfor %}
</div>

<script>
  document.addEventListener('DOMContentLoaded', function() {

    const legendItems = document.querySelectorAll('.legend-item');
    const timelineItems = document.querySelectorAll('.timeline-item');

    const activeCategories = new Set(
      Array.from(legendItems).map(item => item.dataset.category)
    );

    function applyFilter() {
      timelineItems.forEach(function(item) {
        const category = item.dataset.category;
        
        if (activeCategories.has(category)) {
          item.style.display = '';
          setTimeout(() => item.style.opacity = '1', 10);
        } else {
          item.style.opacity = '0';
          setTimeout(() => item.style.display = 'none', 200);
        }
      });

      legendItems.forEach(function(legendItem) {
        const isActive = activeCategories.has(legendItem.dataset.category);
        legendItem.style.opacity = isActive ? '1' : '0.3';
        legendItem.style.textDecoration = isActive ? 'none' : 'line-through';
      });
    }

    legendItems.forEach(function(legendItem) {
      legendItem.addEventListener('click', function() {
        const category = this.dataset.category;

        if (activeCategories.has(category)) {
          activeCategories.delete(category);

          if (activeCategories.size === 0) {
            legendItems.forEach(item => activeCategories.add(item.dataset.category));
          }
        } else {
          activeCategories.add(category);
        }

        applyFilter();
      });
    });

    timelineItems.forEach(function(item) {
      item.style.transition = 'opacity 0.2s ease';
    });

  });
</script>
