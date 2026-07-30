---
name: architecture-diagrams-spider-charts
description: >
  Guide for creating spider charts (progress charts) that monitor architecture
  implementation progress across focus areas or deliverables.
---

# Spider Charts / Progress Charts — Architecture Implementation Monitoring

Spider charts are visual progress tools used in Enterprise Architecture to track
how far implementation has advanced against the 100% completion goal. They are
especially valuable in the Control stage for monitoring architecture deliverables,
focus areas, and implementation maturity.

This skill is for architects who need to surface progress across multiple
architecture topics at a glance and make completion measurable.

---

## When to use this skill

Use this skill when you need to:

- Monitor progress across architecture focus areas or deliverables.
- Compare current state against the ultimate goal of 100% completion.
- Show progress for topics like Organization, Processes, Concepts,
  Applications, and Technology.
- Track the completion of architecture deliverables such as maps, diagrams, or
  repository artifacts.
- Communicate implementation status visually to stakeholders.

---

## Outcome

Produce a Spider Chart or progress chart that shows:

- **Tracked items**: the architecture deliverables, focus areas, or topics.
- **Progress percentages**: the current completion value for each tracked item.
- **Goal comparison**: how each item compares to the 100% target.
- **Visual progress shape**: a connected web showing relative strengths and gaps.

The chart should provide a quick, visual view of architecture implementation
progress and highlight areas that need focus.

---

## Step-by-step workflow

1. **Define the measured items**
   - Choose the deliverables, focus areas, or implementation topics to track.
   - Examples include Organization Map, Business Roles Map, Application Portfolio,
     Information Map, Roadmap, and governance artifacts.

2. **Choose a measurement method**
   - Use a basic method by assigning equal percentage weights to four steps:
     checklist creation, information gathering, deliverable creation, and
     repository documentation.
   - Or use a refined method by assigning percentages based on estimated effort
     or duration for each step.

3. **Calculate progress percentages**
   - For each tracked item, determine the completion percentage using the
     selected method.
   - Ensure the percentages represent the current state vs. the 100% target.

4. **Build the spider chart**
   - Place the tracked items around the chart as radial axes.
   - Use a percentage scale from the center (0%) to the outer edge (100%).
   - Plot the current progress points on each axis.
   - Connect the points to form the web-shaped progress area.

5. **Interpret the chart**
   - Identify which areas are strong and which are lagging.
   - Use the web shape to compare coverage across focus areas.
   - Spot gaps where progress is uneven or insufficient.

6. **Review and update**
   - Validate the percentages with the teams responsible for each deliverable.
   - Update the chart regularly to reflect progress changes.
   - Use the chart in governance reviews and control dashboards.

---

## Decision points and guidance

- **Tracked item selection**: focus on the most meaningful architecture deliverables and implementation areas.
- **Measurement method**: choose the basic method for simplicity or the refined method for greater realism.
- **Number of axes**: limit the chart to a manageable number of items so it remains readable.
- **Visualization**: use distinct labels and consistent percentage scales to keep the chart clear.
- **Use case**: spider charts are best for monitoring progress, not for showing detailed dependencies.

---

## Quality criteria

A high-quality Spider Chart should be:

- **Measurable**: each axis has a defensible percentage value.
- **Comparable**: the chart shows relative progress across items.
- **Readable**: the web shape is easy to interpret and not overcrowded.
- **Validated**: progress values are confirmed by responsible teams.
- **Useful**: it supports governance reviews and highlights control-stage progress.

---

## Example prompt

- "Create a spider chart showing progress for Organization, Processes,
  Information, Applications, and Technology implementation, using percentage
  values for each area."
- "Validate the progress percentages with the architecture team and update the
  chart for the next governance review."

---

## Related guidance

This skill is useful in combination with other architecture artifacts such as:

- Architecture Roadmap
- Work Package Portfolio Map
- Business Process Map
- Information Map
- Application Portfolio Diagram

Use it when you need a visual, progress-focused control metric for architecture
implementation.
