# Security policy

## Reporting a vulnerability

Report suspected vulnerabilities privately, not in a public issue. Use GitHub's private vulnerability reporting on this repository (the Security tab, "Report a vulnerability"), or the contact on https://getskarn.com. Expect an acknowledgement within one business day.

## Scope

This repository is the Skarn plugin for Cursor: two plugin manifests, one hook configuration file, one skill file, and one MCP server declaration. It does not contain the Skarn detection engine, its rules, or the binary. Those are installed separately and maintained on https://getskarn.com, where vulnerabilities in the scanner itself are handled. The hook file here declares which agent events are intercepted and which command is run; it executes nothing on its own. The MCP declaration names a command to spawn; it starts nothing on its own.
