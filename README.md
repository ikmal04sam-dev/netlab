Question 1: Why did the first firewall rule in Step 3 have no effect, and why does blocking h1 → h3 also break h3 → h1?
Why the first rule had no effect:
When the drop rule was added without specifying a priority, it defaulted to the standard priority of 32768. The existing forwarding rule (dl_dst=00:00:00:00:00:03) also had a priority of 32768. When two overlapping flow entries share the same priority, OpenFlow behavior is undefined, and the switch continued matching traffic against the forwarding rule instead of dropping it. Specifying a higher priority (priority=40000) ensured the drop rule was evaluated first.
Why blocking h1 → h3 also breaks h3 → h1:
Network communications rely on two-way traffic. When h3 pings h1, h1 receives the request and generates an ICMP Echo Reply back to h3. The reply packet has a source IP of 10.NN.0.1 (h1) and a destination IP of 10.NN.0.3 (h3). Because the drop rule blocks any ICMP packet where nw_src=10.NN.0.1 and nw_dst=10.NN.0.3, h1's reply is dropped by the switch before reaching h3, causing the ping to fail.

Question 2: In Step 5, why did one conversation survive the controller failure and another didn't?
Surviving conversation (h1 → h2):
Before the controller was killed, h1 and h2 had already communicated. The controller reactively installed explicit forwarding rules for this flow into the switch's flow table with an idle_timeout=60 seconds. As long as traffic continued active within that 60-second window, the switch forwarded packets directly using its cached rules without needing to contact the controller.
Failing conversation (h1 → h3):
Because h1 and h3 had not communicated while the controller was online, no explicit flow rule existed for them in the switch. The switch matched the default table-miss rule (actions=CONTROLLER), which instructed it to send unknown packets to the controller. Since the controller process was dead, the control channel failed, and the switch dropped the unhandled packets.

Question 3: Secure or standalone: which would you choose for a campus network, and what does the other one cost you?
Choice for a Campus Network: Secure Mode
Trade-offs & Costs:
Secure Mode: If the control channel disconnects, the switch drops all packets that lack matching explicit flows rather than failing open. While this risks a network outage if the controller goes down, it guarantees that centralized policies—such as firewall rules, VLAN isolation, and security controls—remain strictly enforced.
Cost of Standalone Mode: In standalone mode, losing the controller causes the switch to fall back to acting like a traditional L2 learning switch. While this maintains basic connectivity, the "cost" is a complete loss of control and security policies: high-priority firewall rules, custom paths, and access controls are bypassed, exposing the campus network to unauthorized traffic and broadcast storms.
