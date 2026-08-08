---
name: project_robotics_automation_direction
description: "Chase's direction on staffing + the robotics/automation future: staffing is a FIXED slider (never modeled as shortage/random variability); service work is moving toward robots + sensors, so the twin's service-resource model must stay resource-agnostic (human OR robot/sensor interchangeable)"
metadata:
  node_type: memory
  type: project
  originSessionId: b5cc1e94-2201-451a-a46f-c770df70e99a
---

**Chase (2026-06-27), on the twin's world-variable build:**

**Staffing = FIXED, sliding input — NOT a world random variable.** Leave it as it already is in the twin (a level you set/slide per scenario). Do NOT model staffing shortages or call-outs as variability. Reasoning (Chase): staffing is an *operations* issue, not a constraint OTTOYARD is solving for — "our business will look bad if we can't staff correctly." So the demo never shows OTTO-Q heroically coping with understaffing; staffing is just a clean dial.

**Strategic direction — robotics/automation handles technician services.** Chase wants to lead into the idea that **robots + sensors** do much of the depot service work (wash, detail, inspection, even confirming/completing a service automatically). A few humans on the ground to begin, but the architecture should let you **interchange a staff member for a robot, or for sensors that auto-confirm or auto-complete a service.** 

**How to apply (downstream):** keep the twin's **service-resource model resource-agnostic** — a "service resource" / "service completion" should not be hard-coded as human. A bay's work can be done (and confirmed) by a human tech OR a robot/sensor; service *durations* are world facts (a wash takes time regardless), but a robot resource can later carry tighter/faster/more-consistent duration distributions + auto-confirmation (no human sign-off step). This ties to the "gigantic ability" Chase wants on the OTTO-Q backend: OTTO-Q orchestrating a mostly-automated depot. Don't build human-staffing dependencies that would block swapping in robotic/sensor resources later. See [[project_depot_ops_model]], [[project_twin_variable_backend]], [[project_ottoq_logic_completeness]].
