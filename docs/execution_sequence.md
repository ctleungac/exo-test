# EXO CLI execution sequence

The diagram below traces how the `exo` console entry point dispatches work across modules. It reflects the default path where the CLI starts the node, GRPC server, discovery, and optional model execution.

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant CLI as "exo (console script)"
    participant Run as "exo.main.run()"
    participant EventLoop as "configure_uvloop()"
    participant Main as "exo.main.main()"
    participant Node as "orchestration.node.Node"
    participant Server as "networking.grpc.GRPCServer"
    participant Discovery as "networking.<discovery>"
    participant API as "api.ChatGPTAPI"

    User->>CLI: invoke `exo` command
    CLI->>Run: console_scripts entry point
    Run->>EventLoop: configure_uvloop()
    EventLoop-->>Run: event loop configured
    Run->>Main: loop.run_until_complete(main())
    Main->>Main: argparse parses command/flags
    Main->>Main: create ShardDownloader and inference engine
    Main->>Discovery: instantiate UDP/Tailscale/Manual discovery
    Main->>Node: construct Node(inference_engine, discovery,...)
    Main->>Server: construct GRPCServer(node, host, port)
    Main->>API: construct ChatGPTAPI(node,...)
    Main->>Node: register topology/token callbacks
    Main->>Node: await node.start(wait_for_peers)
    Node->>Server: start()
    Node->>Discovery: start() and peer discovery
    Node-->>Main: ready for workload
    alt `run`/`--run-model`
        Main->>Node: run_model_cli(...) -> process_prompt()
        Node->>Server: broadcast/send tokens
    else `eval`/`train`
        Main->>Node: eval_model_cli/train_model_cli
        Node->>Node: coordinate inference/training
    else default
        Main->>API: api.run(chatgpt_api_port)
    end
```
