# Final Presentation Guide

- **Length:** 15 minutes of talk, then 5 minutes of Q&A. The chair stops talks at 16 minutes.
- **Slides:** 12–18 slides. Use the shared MINDFRAME slide master (HUB provides it).
- **Audience:** The team, supervisors, and guests. Assume that the audience knows robotics but does not know your subsystem.

## Slide structure

| # | Slide | Time | Content |
|---|-------|------|---------|
| 1 | Title | 0:15 | Project name, subsystem, your name, role |
| 2 | Where I fit | 1:00 | MINDFRAME architecture diagram with your box highlighted; your inputs/outputs |
| 3 | Problem | 1:00 | What must work for the cup task; why it is hard on a humanoid |
| 4 | Targets | 0:45 | RFC acceptance criteria table |
| 5–7 | Approach | 3:00 | Design, key decisions with alternatives, one diagram per slide |
| 8 | Demo video | 2:00 | 1–3 min clip, real robot preferred |
| 9–10 | Results | 2:30 | Target vs. measured table; one key plot |
| 11 | Failures | 1:30 | Top failure modes with examples and causes |
| 12 | Integration | 1:00 | How your subsystem performed in the end-to-end task |
| 13 | Lessons learned | 1:00 | 3 points |
| 14 | Next steps | 0:45 | Top 3 future improvements |
| 15 | Backup slides | — | Extra details for Q&A (not counted in the time) |

## Rules

1. Show measured numbers with the number of trials (for example, "87% (26/30)").
2. Use one main message per slide, and put that message in the slide title.
3. Put a pre-recorded video on a slide. Do not depend on the live robot during your individual talk. The live demo happens at the end of the day.
4. Rehearse once with your backup partner, and once in the team rehearsal in Week 26.
5. Embed videos in the file and keep a copy on a USB drive.

## Subsystem-specific must-show items

| Project | Must show |
|---------|-----------|
| CORTEX | A live or recorded instruction → generated plan → execution trace; plan validity and planning latency; one recovery example |
| IRIS | Detection overlay video; 3D position error plot; a hard case such as a transparent cup |
| ATLAS | The 3D map and the labeled semantic map; localization error over a walking path |
| STRIDE | Top-down path video; docking accuracy distribution; obstacle stop test |
| GRIP | Pick and place video; grasp success by cup type; a failed grasp with the cause |
| SPINE | E-stop latency measurement; mode state machine; the safety monitor log |
| DOJO | The simulation scene; benchmark trends over the project; sim-vs-real comparison table |
| HUB | Operator app walkthrough; deploy time; data recorded (hours, GB, number of runs) |
