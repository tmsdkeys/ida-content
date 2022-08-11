# Adding Packet and Acknowledgement data

In this section we learn how to define packet and acks (acknowledgements) for the Leaderboard blockchain. Remember that this blockchain will mostly be receiving packets from the checkers blockchain or other gaming chains. This will be handled in the checkers blockchain extension tutorial. In this section we will add an additional packet definition that will enable the Leaderboard chain to send a packet to connected game chains when a player has entered the top of the rankings.

The documentation on how to define packet and acks in IBC can be found in [the IBC go docs](https://ibc.cosmos.network/main/ibc/apps/packets_acks.html).

## Scaffold a packet with Ignite CLI

We are now going to be scaffolding the IBC packet data with Ignite CLI and compare once more with git diff:

```bash
ignite scaffold packet ibcTopRank playerId rank score --ack playerId --module leaderboard
```

Note that the packet is called `ibcTopRank`, which includes the fields `playerId`, `rank` and `score`. Additionally we could send back the `playerId` of the player who entered the top of the rankings through the `Acknowledgement`.

We see the output on the terminal that gives an overview of the changes made:

```bash
modify proto/leaderboard/packet.proto
modify proto/leaderboard/tx.proto
modify x/leaderboard/client/cli/tx.go
create x/leaderboard/client/cli/tx_ibc_top_rank.go
modify x/leaderboard/handler.go
create x/leaderboard/keeper/ibc_top_rank.go
create x/leaderboard/keeper/msg_server_ibc_top_rank.go
modify x/leaderboard/module_ibc.go
modify x/leaderboard/types/codec.go
modify x/leaderboard/types/events_ibc.go
create x/leaderboard/types/messages_ibc_top_rank.go
create x/leaderboard/types/messages_ibc_top_rank_test.go
create x/leaderboard/types/packet_ibc_top_rank.go

🎉 Created a packet `ibcTopRank`.
```
In the next paragraphs we will investigate each of the most important additions to the code.

## Proto definitions

The first additions are to the proto definitions in the `packet.proto` and `tx.proto` files.

```diff
@@ message LeaderboardPacketData {
    oneof packet {
        NoData noData = 1;
        // this line is used by starport scaffolding # ibc/packet/proto/field
+		IbcTopRankPacketData ibcTopRankPacket = 2; 
        // this line is used by starport scaffolding # ibc/packet/proto/field/number
    }
}
```
with `IbcTopRankPacketData`:
```protobuf
// IbcTopRankPacketData defines a struct for the packet payload
message IbcTopRankPacketData {
  string playerId = 1;
  string rank = 2;
  string score = 3;
}
```
and the ack:
```protobuf
// IbcTopRankPacketAck defines a struct for the packet acknowledgment
message IbcTopRankPacketAck {
	  string playerId = 1;
}
```

And in `tx.proto` a Msg service is added:
```diff
// Msg defines the Msg service.
service Msg {
+      rpc SendIbcTopRank(MsgSendIbcTopRank) returns (MsgSendIbcTopRankResponse);
    // this line is used by starport scaffolding # proto/tx/rpc
}
```

where: 

```protobuf
message MsgSendIbcTopRank {
  string creator = 1;
  string port = 2;
  string channelID = 3;
  uint64 timeoutTimestamp = 4;
  string playerId = 5;
  string rank = 6;
  string score = 7;
}

message MsgSendIbcTopRankResponse {
}
```

**Note**: the proto definitions will be compiled into `types/packet.pb.go` and `types/tx.pb.go`.

## CLI commands

Ignite CLI also creates CLI commands to send packets and adds them to the `client/cli/` folder.

We can thus send packets from the CLI with the following command:
```bash
leaderboardd tx leaderboard send-ibcTopRank [portID] [channelID] [playerId] [rank] [score]
```

## Module's keeper

TODO

## Packet callbacks

TODO

