```mermaid
flowchart TD
    %% ============================================================
    %% 1. MESSAGE META CREATION
    %% ============================================================
    subgraph STEP1["MESSAGE META CREATION
    
    <br/><br/>"]
        A1["MessageMeta Controller<br/>.create()"]
        A2["MessageMeta Controller<br/>.approve()"]
        A3["MessageMetaConfig<br/>(MongoDB)"]
        A4["Status = APPROVED<br/>(MongoDB)"]

        A1 --> A3
        A2 --> A4
        A1 --> A2
    end

    %% ============================================================
    %% 2. SEND REQUEST
    %% ============================================================
    subgraph STEP2["SEND REQUEST<br/><br/>"]
        B1["DemoController<br/>.sendTxnMsg()"]
        B2["Transactional MessageProcessor"]
        B3["Transactional MessageFacade<br/>.sendMessage()"]

        B1 --> B2 --> B3
    end

    %% ============================================================
    %% 3. TAG RESOLUTION (VELES)
    %% ============================================================
    subgraph STEP3["TAG RESOLUTION (VELES)<br/><br/>"]
        C1["RecipientTag Resolver"]
        C2["Extract Tags<br/>(Liquid / Simple)"]
        C3["Build UserContext<br/>Execute Actions"]
        C4["Load Tags via BaseTagLoader"]

        C1 --> C2 --> C3 --> C4
    end

    %% ============================================================
    %% 4. DETEMPLATIZATION
    %% ============================================================
    subgraph STEP4["DETEMPLATIZATION<br/><br/>"]
        D1["Detemplatize MessageProcessor"]
        D2["NsadminMessage Builder Factory"]
        D3["Channel-Specific Message Builder<br/>(Email / SMS / etc)"]
        D4["Detemplatization<br/>(Liquid / Simple)"]

        D1 --> D2 --> D3 --> D4
    end

    %% ============================================================
    %% 5. NSADMIN QUEUE
    %% ============================================================
    subgraph STEP5["NSADMIN QUEUE<br/><br/>"]
        E1["NsadminQueue Service"]
        E2["NSADMIN_RECIPIENTS_LIST_TRANS<br/>(RabbitMQ)"]

        E1 --> E2
    end

    %% ============================================================
    %% 6. FINAL DELIVERY
    %% ============================================================
    subgraph STEP6["FINAL DELIVERY<br/><br/>"]
        F1["NSAdmin Service"]
        F2["Gateway Delivery<br/>(SMS / Email / etc)"]

        F1 --> F2
    end

    %% ============================================================
    %% FLOW CONNECTIONS BETWEEN STEPS
    %% ============================================================
    STEP1 --> STEP2
    STEP2 --> STEP3
    STEP3 --> STEP4
    STEP4 --> STEP5
    STEP5 --> STEP6
