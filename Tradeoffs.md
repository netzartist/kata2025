# Trade-Offs

## Robustness of delivery processes

Our core business is providing vehicles for rent in a best-effort fashion. When introducing things like food delivery services we also need proper error handling to prevent situations where deliveries get "forgotten" by the system. Therefore, an event-driven system might increase the risk of loosing task overview. This is especially critical for our tourist costumers since they heavily rely on arrival-day deliveries. Leaving them without promised delivery would cause substential reputation damage.

## LLM-based food requests

Even though it might be easy for the system to "understand" what the user wants, it might be more difficult to map this to available goods in local stores. Depending on the user's wishes it might be hard or impossible to find the requested article. Ideally, those cases should be identified as early as possible and somehow prevented by the system. This might require either a more currated and restricted list of possible goods (loss of flexibility) or an uplink to the respective local food store (increase of complexity).
