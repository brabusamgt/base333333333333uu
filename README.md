# base333333333333uu
0x2Fa54408f14C05d074463cFcBb4023A014D460b1
import time
from collections import defaultdict
import networkx as nx
from web3 import Web3

RPC_URL = "https://mainnet.base.org"

TRANSFER_TOPIC = Web3.keccak(
    text="Transfer(address,address,uint256)"
).hex()

WINDOW_BLOCKS = 50

ZERO = "0x0000000000000000000000000000000000000000"


def decode_address(topic):
    return "0x" + topic.hex()[-40:]


def main():

    w3 = Web3(Web3.HTTPProvider(RPC_URL))

    if not w3.is_connected():
        raise RuntimeError("Cannot connect")

    print("Connected to Base")
    print("Computing wallet centrality...\n")

    last_block = w3.eth.block_number

    graph = nx.DiGraph()

    while True:

        try:

            current_block = w3.eth.block_number

            if current_block >= last_block + WINDOW_BLOCKS:

                logs = w3.eth.get_logs({
                    "fromBlock": current_block - WINDOW_BLOCKS,
                    "toBlock": current_block,
                    "topics": [TRANSFER_TOPIC]
                })

                edge_weights = defaultdict(int)

                for log in logs:

                    sender = decode_address(log["topics"][1])
                    receiver = decode_address(log["topics"][2])

                    if sender == ZERO or receiver == ZERO:
                        continue

                    edge_weights[(sender, receiver)] += 1

                graph.clear()

                for (sender, receiver), weight in edge_weights.items():
                    graph.add_edge(
                        sender,
                        receiver,
                        weight=weight
                    )

                if graph.number_of_nodes() > 0:

                    centrality = nx.pagerank(
                        graph,
                        weight="weight"
                    )

                    ranking = sorted(
                        centrality.items(),
                        key=lambda x: x[1],
                        reverse=True
                    )[:20]

                    print("\n===== Top Central Wallets =====\n")

                    for wallet, score in ranking:

                        print(wallet)
                        print(
                            "PageRank:",
                            round(score, 6)
                        )
                        print(
                            "Incoming:",
                            graph.in_degree(wallet)
                        )
                        print(
                            "Outgoing:",
                            graph.out_degree(wallet)
                        )
                        print()

                last_block = current_block

            time.sleep(3)

        except Exception as e:

            print("Error:", e)
            time.sleep(5)


if __name__ == "__main__":
    main()
