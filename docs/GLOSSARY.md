# Glossary

Use these terms with these meanings in all documents and code.

| Term | Meaning |
|------|---------|
| **Subsystem** | One of the 8 member projects (CORTEX, IRIS, ATLAS, STRIDE, GRIP, SPINE, DOJO, HUB). Each subsystem is one package and one service. |
| **Contract** | A versioned interface definition in `contracts/`: either a protobuf service or a stream entry in `streams.yaml`. |
| **Skill** | A long-running robot capability that CORTEX can call: `NavigateTo`, `FindObjects`, `Pick`, `Place`, `LookAt`, `Speak`. |
| **Mock** | A fake implementation of a contract that returns scripted or simple results. Used to develop without the real subsystem. |
| **Backend** | The thing a RobotService talks to: the real G1 (SPINE) or the simulator (DOJO). |
| **Place** | A named location in the world model, for example `kitchen_table`. It has a pose, a room, and approach poses. |
| **Object instance** | A detected object with an ID, a label, a pose, and a last-seen time. |
| **Result card** | The JSON and Markdown output of a benchmark run (Engineering Standards §5). |
| **Verify** | `just verify`: the one command that reproduces and checks a subsystem result. |
| **Profile** | A named configuration set: `sim`, `replay`, `robot_safe`, `robot`. |
| **Replay** | Running a subsystem on a recorded MCAP file instead of live sensors. |
| **Gate** | A milestone review with fixed pass criteria. |
| **ADR** | Architecture Decision Record: a short document that records one technical decision. |
| **E-stop** | Emergency stop. Hardware (the remote controller) or software (`RobotService.EStop`). |
| **PC2** | The Jetson development computer inside the G1 that runs our robot-side services. |
| **Workstation** | The off-board x86 GPU computer that runs heavy models. |
