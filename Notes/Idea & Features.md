# General Idea
***Procpm (Process Package Manager)*** is a developer tool that manages an entire software project and all of the programs / services that belong to it.

It's a combination of
-  Project Manager (Think of *uv*)
-  A terminal
-  A service / process manager (see what is running)
-  A log viewer
-  A system monitor
-  A developer dashboard (moves under a terminal stuff)
-  A plugin system

Instead of opening several terminals and running different processes like:

```bash
python backend.py # Takes up one terminal

npm run dev # Takes up another terminal

python worker.py # Takes up another terminal...
```

You can just type:
```bash
procpm start
```

and all the scripts you want to run will run.

Procpm will also keep track of what crashes.

```bash
procpm status
```

You would end up seeing something like

```bash
ProcPM

Service      Status      Port      CPU     RAM   
-------------------------------------------------
Backend      Running     8000      3.2%    120 MB
Frontend     Running     3000      1.4%    80 MB
Worker       Running     --        2.1%    50 MB
Database     Running     5432      0.8%    200 MB
```

A Procpm file auto generates a configuration you can edit, for example:
```toml
[project]
name = "my-project"

[services.backend]
command = "python server.py"
port = 8000

[services.frontend]
command = "npm run dev"
port = 3000
depends_on = ["backend"]

[services.worker]
command = "python worker.py"
depends_on = ["backend"]
```

The configuration file is read and Procpm knows
-  What programs need to be ran
-  How to start them
-  What order they should start in
-  What programs depend on other programs
-  Which ports they use
-  How to restart them
-  Where their logs go
-  How to monitor them

## What Can Procpm Do?
### Create Projects
```bash
procpm init my-project
```

Example Project Layout:
```bash
my-project/
|--- procpm.toml
|--- .procpm/
     |--- state.db (State of processes running)
     |--- logs/
     |--- cache/
```

### Start / Stopping services
`procpm start` starts all the services in the `procpm.toml` file. You can start a service individually with `procpm start <service_name>`.

As well as starting, you can stop services with `procpm stop` or `procpm stop <service_name>`.

Services need to be defined in the `procpm.toml` file.

### Restarting Services
`procpm restart` or `procpm restart <service_name>` can restart already running services, whether code broke, code changes, or just because you want to.

### Monitor Processes
Procpm keeps track of:
- Whether a service is running
- Process ID
- CPU usage
- RAM usage
- How long it has been running
- Whether it crashed
- How many times it restarted
- Exit codes

Example:
```bash
<Service_name>
Status: Running
PID: 18242
CPU: 3.2%
RAM: 120 MB
Uptime: 24 Minutes
Restarts: 0
```

### Automatically restart crashed programs
You could configure:
```toml
restart = "on-failure"
```

If the program crashes, Procpm notices and starts it again.

It should also have limits so a broken program doesn't constantly restart forever.

For example:

```bash
Backend crashed
Restarting... 1/5

Backend crashed
Restarting... 2/5

Backend crashed
Restarting... 3/5
```
After too many failures:
```bash
Backend permanently failed.
Maximum restart attempts reached.
```

### Services Depend on each other
You can specify a start order for services, where if service A depends on service B, service B starts before service A.

Also Procpm will detect if there are any circular dependencies and it will report it.

### Every Service Produces Logs
Instead of opening different terminals, Procpm collected them.

Example:
```bash
.procpm/
|--- logs/
     |--- backend.log
     |--- frontend.log
     |--- worker.log
```

Or you could use
```bash
nexus logs backend

nexus logs backend --follow # Follow new logs as they appear
```

### SQLite Database
Procpm uses SQLite to remember information

It could store:
- Projects
- Services
- Previous processes
- Start/stop times
- Crashes
- Restart counts
- Events
- Plugins
- Configuration versions

### Interface
Procpm should have CLI Commands *(with rich output)* and a TUI *(Textual)* interface.

### Plugins and an API
Procpm may include an API where you could get data or post commands. Also plugins may be included *(more info later)*.

### Doctor
Procpm (`procpm doctor`) would be a diagnostic command.

Running `procpm doctor` could produce:
```bash
PROCPM DOCTOR

[OK] Configuration
[OK] Database
[OK] Python
[OK] Backend command
[WARN] Port 3000 already in use
[OK] Plugin System
[FAIL] Frontend command not found
```

### Installation
Procpm should be installed like a normal python program.

```bash
pip install procpm
```

Then
```bash
procpm help # Leads to Online Docs
```