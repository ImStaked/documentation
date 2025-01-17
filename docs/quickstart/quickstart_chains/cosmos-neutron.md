# Neutron Quick Start

## Goals

The goal of this quick start guide is to index all [airdrop claims](https://www.mintscan.io/neutron/wasm/contract/neutron198sxsrjvt2v2lln2ajn82ks76k97mj72mtgl7309jehd0vy8rezs7e6c56) on [Neutron Network](https://www.mintscan.io/neutron/).

::: info
Neutron Network is a chain based on the Cosmos SDK. Therefore you can index chain data via the standard Cosmos RPC interface.

Before we begin, make sure that you have initialised your project using the provided steps in the **[Start Here](../quickstart.md)** section. You must complete the suggested [4 steps](https://github.com/subquery/cosmos-subql-starter#readme) for Cosmos users.
:::

In every SubQuery project, there are 3 key files to update. Let's begin updating them one by one.

::: tip
The final code of this project can be found [here](https://github.com/subquery/cosmos-subql-starter/tree/main/Neutron/neutron-starter).
:::

## 1. Your Project Manifest File

The Project Manifest (`project.ts`) file is an entry point to your project. It defines most of the details on how SubQuery will index and transform the chain data. For Cosmos chains, there are four types of mapping handlers (and you can have more than one in each project):

- [BlockHanders](../../build/manifest/cosmos.md#mapping-handlers-and-filters): On each and every block, run a mapping function
- [TransactionHandlers](../../build/manifest/cosmos.md#mapping-handlers-and-filters): On each and every transaction, run a mapping function
- [MessageHandlers](../../build/manifest/cosmos.md#mapping-handlers-and-filters): On each and every message that matches optional filter criteria, run a mapping function
- [EventHanders](../../build/manifest/cosmos.md#mapping-handlers-and-filters): On each and every event that matches optional filter criteria, run a mapping function

Note that the manifest file has already been set up correctly and doesn’t require significant changes, but you need to change the datasource handlers. This section lists the triggers that the manifest file looks for on the blockchain to start indexing.

```ts
{
  dataSources: [
    {
      kind: SubqlCosmosDatasourceKind.Runtime,
      startBlock: 1,
      mapping: {
        file: "./dist/index.js",
        handlers: [
          {
            handler: "handleAirdropClaim",
            kind: SubqlCosmosHandlerKind.Message,
            filter: {
              type: "/cosmwasm.wasm.v1.MsgExecuteContract",
              contractCall: "claim",
              values: {
                contract:
                  "neutron198sxsrjvt2v2lln2ajn82ks76k97mj72mtgl7309jehd0vy8rezs7e6c56",
              },
            },
          },
        ],
      },
    },
  ],
}
```

The above code defines that you will be running one handler: A `handleAirdropClaim` message handler which will be triggered when a `claim` message is encountered on a `MsgExecuteContract` type. The `contract` value is the address of the neutron airdrop contract.

Check out our [Manifest File](../../build/manifest/cosmos.md) documentation to get more information about the Project Manifest (`project.ts`) file.

## 2. Update Your GraphQL Schema File

The `schema.graphql` file determines the shape of the data that you are using SubQuery to index, hence it's a great place to start. The shape of your data is defined in a GraphQL Schema file with various [GraphQL entities](../../build/graphql.md).

Update the `schema.graphql` file as follows. In this project, since we are indexing all [airdrop claims](https://www.mintscan.io/neutron/wasm/contract/neutron198sxsrjvt2v2lln2ajn82ks76k97mj72mtgl7309jehd0vy8rezs7e6c56) on Neutron, we have a `Claim` entity that includes a number of properties, including transaction hash and block data as well as date, amount and receiver data.

Daily aggregated claims are also captured in a seperate `DailyClaimSummary` entity.

```graphql
type Claim @entity {
  id: ID! # Index
  transactionHash: String!
  blockHeight: BigInt!
  date: Date!
  receiver: String!
  amount: BigInt!
}

type DailyClaimSummary @entity {
  id: ID! # this is the ISO string of the day e.g. '2023-03-26'
  total_claimed: BigInt!
}
```

::: warning Important
When you make any changes to the schema file, do not forget to regenerate your types directory.
:::

::: code-tabs
@tab:active yarn

```shell
yarn codegen
```

@tab npm

```shell
npm run-script codegen
```

:::

You will find the generated models in the `/src/types/models` directory.

As you're creating a new CosmWasm based project, this command will also generate types for your listed protobufs and save them into `src/types` directory, providing you with more typesafety. Read about how this is done in [Cosmos Codegen from CosmWasm Protobufs](../../build/introduction.md#cosmos-codegen-from-cosmwasm-protobufs).

Check out our [GraphQL Schema](../../build/graphql.md) documentation to get more information on `schema.graphql` file.

Now that you have made essential changes to the GraphQL Schema file, let’s go ahead with the next configuration.

## 3. Add a Mapping Function

Mapping functions determine how chain data is transformed into the optimised GraphQL entities that you previously defined in the `schema.graphql` file.

Navigate to the default mapping function in the `src/mappings` directory and update your mapping files to match the following (**note the additional imports**):

```ts
import { CosmosMessage } from "@subql/types-cosmos";
import { Claim, DailyClaimSummary } from "../types";

type AirdropClaimMessageType = {
  type: string;
  sender: string;
  contract: string;
  msg: {
    claim: {
      amount: string;
      proof: string[];
    };
  };
};

async function checkGetDailyClaim(date: Date): Promise<DailyClaimSummary> {
  // Create the ID from the iso date string (e.g. '2023-03-26')
  // Timestamps are in seconds, need to convert to ms
  const id = date.toISOString().substring(0, 10);
  // Read to see if there is an existing aggregation record
  let dailyClaimSummary = await DailyClaimSummary.get(id);
  if (!dailyClaimSummary) {
    // This is a new day! Create a new aggregation
    dailyClaimSummary = DailyClaimSummary.create({
      id,
      total_claimed: BigInt(0),
    });
  }
  return dailyClaimSummary;
}

export async function handleAirdropClaim(
  msg: CosmosMessage<AirdropClaimMessageType>
): Promise<void> {
  // Example https://www.mintscan.io/neutron/txs/156FE31585BD75E06EE337CEA908C37EA0434CC49943B4860E7AABE2475B6B01?height=1437614
  logger.info(
    `New Airdrop Claim at block ${msg.block.header.height.toString()}`
  );

  // Claim records are created from on chain data
  const airdropClaimRecord = Claim.create({
    id: `${msg.tx.hash}-${msg.idx}`,
    blockHeight: BigInt(msg.block.block.header.height),
    date: new Date(msg.block.header.time.toISOString()),
    transactionHash: msg.tx.hash,
    receiver: msg.msg.decodedMsg.sender,
    amount: BigInt(msg.msg.decodedMsg.msg.claim.amount),
  });

  // We update the daily aggregation
  const dailyClaimSummary = await checkGetDailyClaim(airdropClaimRecord.date);
  dailyClaimSummary.total_claimed += airdropClaimRecord.amount;

  // Save data to the store
  await dailyClaimSummary.save();
  await airdropClaimRecord.save();
}
```

Here we have two functions. Our `handleAirdropClaim` handler function which was defined in the manifest file along with an extra function called `checkGetDailyClaim`.

`handleAirdropClaim` receives a message of type CosmosMessage, logs a message to the console for debugging purposes and then attempts to obtain the records from the `msg` attributes. The transaction hash along with the message id is used as a unique id.

An airdrop claim object is then created, provided that it hasn't been created already, and then we index certain message attributes depending on the data we need.

The `checkGetDailyClaim` function is called by the previous function to determine the total quantity of claims made during the day. It is called when each new claim object is created or updated.

Check out our [Mappings](../../build/mapping/cosmos.md) documentation and get information on the mapping functions in detail.

## 4. Build Your Project

Next, build your work to run your new SubQuery project. Run the build command from the project's root directory as given here:

::: code-tabs
@tab:active yarn

```shell
yarn build
```

@tab npm

```shell
npm run-script build
```

:::

::: warning Important
Whenever you make changes to your mapping functions, you must rebuild your project.
:::

Now, you are ready to run your first SubQuery project. Let’s check out the process of running your project in detail.

## 5. Run Your Project Locally with Docker

Whenever you create a new SubQuery Project, first, you must run it locally on your computer and test it and using Docker is the easiest and quickest way to do this.

The `docker-compose.yml` file defines all the configurations that control how a SubQuery node runs. For a new project, which you have just initialised, you won't need to change anything.

However, visit the [Running SubQuery Locally](../../run_publish/run.md) to get more information on the file and the settings.

Run the following command under the project directory:

::: code-tabs
@tab:active yarn

```shell
yarn start:docker
```

@tab npm

```shell
npm run-script start:docker
```

:::

::: tip Note
It may take a few minutes to download the required images and start the various nodes and Postgres databases.
:::

## 6. Query your Project

Next, let's query our project. Follow these three simple steps to query your SubQuery project:

1. Open your browser and head to `http://localhost:3000`.

2. You will see a GraphQL playground in the browser and the schemas which are ready to query.

3. Find the _Docs_ tab on the right side of the playground which should open a documentation drawer. This documentation is automatically generated and it helps you find what entities and methods you can query.

Try the following queries to understand how it works for your new SubQuery starter project. Don’t forget to learn more about the [GraphQL Query language](../../run_publish/query.md).

```graphql
{
  query {
    claims(first: 5, orderBy: AMOUNT_DESC) {
      totalCount
      nodes {
        id
        transactionHash
        blockHeight
        date
        receiver
        amount
      }
    }
    dailyClaimSummaries(first: 5, orderBy: ID_DESC) {
      nodes {
        id
        totalClaimed
      }
    }
  }
}
```

You will see the result similar to below:

```json
{
  "data": {
    "query": {
      "claims": {
        "totalCount": 29,
        "nodes": [
          {
            "id": "0BE48D29E571443DCBB69D166242EC0D82500CA574BD18B5EC4A2FC9B4F4DA52-0",
            "transactionHash": "0BE48D29E571443DCBB69D166242EC0D82500CA574BD18B5EC4A2FC9B4F4DA52",
            "blockHeight": "1437800",
            "date": "2023-07-13T14:56:15.706",
            "receiver": "neutron13yl8ksc640t4zj9ur2q0e9hcxuphxkfm9v7an9",
            "amount": "1126628770"
          },
          {
            "id": "E11FEDF7A703F8A01DB5B9251D22797AAC292F1FDC5B96EA8C378E9249B9B0FB-0",
            "transactionHash": "E11FEDF7A703F8A01DB5B9251D22797AAC292F1FDC5B96EA8C378E9249B9B0FB",
            "blockHeight": "1438551",
            "date": "2023-07-13T15:39:28.097",
            "receiver": "neutron1rmemz0vyxjzqgy4hljymvzq3e3uf46303wl7lv",
            "amount": "806706762"
          },
          {
            "id": "B75398DD51A3CB655908BFB9352A3F1BDE0910D095B8EB1D35E3817A3BE7D896-0",
            "transactionHash": "B75398DD51A3CB655908BFB9352A3F1BDE0910D095B8EB1D35E3817A3BE7D896",
            "blockHeight": "1438402",
            "date": "2023-07-13T15:30:36.123",
            "receiver": "neutron1h4j8r3wzf5lkkvvdnu9fulh7qpqpwu6ufeh56w",
            "amount": "733135188"
          },
          {
            "id": "7A926359E74A04FE929488C10A298F325CCF6A3BEC4D6867CF375F972D1C085E-0",
            "transactionHash": "7A926359E74A04FE929488C10A298F325CCF6A3BEC4D6867CF375F972D1C085E",
            "blockHeight": "1438955",
            "date": "2023-07-13T16:02:38.373",
            "receiver": "neutron10quf7r9jejsv5ahshc0z0rlfmxccy7kxaxxwy0",
            "amount": "113793918"
          },
          {
            "id": "2D857594AF05D3F6508BB7854B3253B5F4F9C0E27ECE888E8FDEF5DB59746F06-0",
            "transactionHash": "2D857594AF05D3F6508BB7854B3253B5F4F9C0E27ECE888E8FDEF5DB59746F06",
            "blockHeight": "1439106",
            "date": "2023-07-13T16:11:02.756",
            "receiver": "neutron1wj3y6eqjd5c8jcfd25w38jkjd66455faj92f4l",
            "amount": "113334003"
          }
        ]
      },
      "dailyClaimSummaries": {
        "nodes": [
          {
            "id": "2023-07-13",
            "totalClaimed": "3899514263"
          }
        ]
      }
    }
  }
}
```

## What’s Next?

Congratulations! You have now a locally running SubQuery project that accepts GraphQL API requests for transferring data from bLuna.

::: tip Tip

Find out how to build a performant SubQuery project and avoid common mistakes in [Project Optimisation](../../build/optimisation.md).

:::

Click [here](../../quickstart/whats-next.md) to learn what should be your **next step** in your SubQuery journey.
