---
layout: ModuleLandingPage
main: true
order: 0
intro:
  - overline: Developer Course
    title: Interchain Developer Academy
    image: /graphics-sdk-course.png
    description: |
      Welcome to the Interchain Developer Academy!<br/><br/>
      Over the next seven weeks, you will dive deep into the Interchain Ecosystem. Let's get started!
    action:
      label: Start learning!
      url: /ida-course/LPs/week-0/
overview:
  title: Important program information
  items:
    - title: Timeline and deadlines
      description: |
        Academy start: May 18th<br/><br/>
        Program duration: May 18th, 2023 until July 6th, 2023 (seven weeks)<br/><br/>
        Exam period: July 6th, 2023 to August 4th, 2023<br/><br/>
        Results available: August 18th, 2023<br/><br/>
    - title: What you will learn
      description: |
        Over the next seven weeks, you will dive deep into the Interchain Ecosystem, starting with a high-level introduction to familiarize yourself with the main concepts. Next, you will put theory into practice by learning how to initiate and build an application-specific blockchain using the Cosmos SDK; how to use Ignite CLI to scaffold modules for your blockchain; and how to connect a chain with other chains using the Inter-Blockchain Communication Protocol (IBC). You will learn how to build frontend and backend applications using CosmJS; operate nodes and validate on an Interchain blockchain; and run a relaying infrastructure between IBC-connected chains.
    - title: How to get the most out of the Academy
      description: |
          The Academy is self-paced and flexible, so you do not have to be online at particular times. You can follow the weekly plan or go through the learning material at your own pace. We recommend allocating about 10 to 15 hours a week to get through all the material.<br/><br/>
          The material is delivered in various formats, including text, images, videos, quizzes, and exercises. There is plenty of additional material embedded in the content to deepen your understanding of particular concepts. And if you want even more, ask your tutors and expert instructors, who will point you in the right direction!<br/><br/>
          <b>Hands-on exercises</b><br/><br/>
          In each module, you will find quizzes, code exercises, and/or code examples. In the first two weeks, you can find a quiz (end of Week 1) and an exercise (end of Week 2). It does not matter if you pass the quiz or exercise - think of these as opportunities to practice and demonstrate your engagement with the program. Both will remain open until the launch of the final exam, but we recommend taking them as soon as you finish Week 1 and 2.<br/><br/>
          Week 1: Quiz - recommended date: Thursday, May 25th<br/><br/>
          Week 2: Exercise - recommended date: Thursday, June 1st<br/><br/>
          Week 1 Quiz & Week 2 Exercise - <b>closing date</b>: Thursday, July 6th<br/><br/>
          You will get the results of submitted exercises.<br/><br/>
          <b>Technical requirements</b><br/><br/>
          No special technical requirements of HW or SW are needed. You need a computer with at least 8 GB RAM and 4 GB free hard disk space.
    - title: How much time do I need to dedicate to the Academy?
      description: |
        There are roughly 80 hours of learning material and exercises to work through. In addition, you need to plan for about 20 hours to complete the final exam. In our experience, participants who allocate about 10 to 15 hours of work per week tend to get the most out of the program and perform best. However, learning styles are different, so work at a pace that suits you!<br/><br/>
        All the materials are available right from the start of the program.
    - title: What support will I get in the Academy? 
      description: |
        We have set up a private Discord for the Academy for all teaching and ongoing communication. You can reach out to your instructors anytime for support. We encourage you to proactively collaborate with other participants in your cohort and with your instructors. Ask questions, request feedback, and seek help if you are stuck! That way, you will get the most out of the Academy.<br/><br/>
        We aim to answer your questions within a few hours. Our maximum response time is 24 hours. Main support hours are on weekdays between 6 AM UTC and 4 PM UTC. We do not provide support during the weekends.<br/><br/>
        Click <a href="/ida-course/discord-info.html">here</a> to learn how to join and use Discord.<br/><br/>
        You will get detailed information on how to join and use Discord via email.
    - title: How do I access Discord?
      description: |
        Follow these two steps to join the private Academy channels on Discord:<br/><br/>
        1. Join the official Cosmos Discord by clicking <a href="https://discord.gg/cosmosnetwork">here</a>.Follow the verification process. It is straightforward but if you need guidance, read <a href="https://medium.com/@alicemeowuk/cosmos-developers-discord-access-7c15951cc839">this article</a>.<br/><br/>
        2. After joining the Discord server, go <a href="https://academy.interchain.io/onboarding/?token=%7B$b9_uuid%7D">here</a> and enter your Discord ID. You will automatically be added to the Discord area for participants called "Interchain Developer Academy".<br/><br/>
        If you have any problems, email us at <a href="mailto:academy@interchain.io">academy@interchain.io</a>.<br/><br/>
        We have put together a <a href="/ida-course/discord-info.html">quick guide</a> explaining how to best communicate on Discord.
    - title: How do I get certified?
      description: |
        After the seven-week program, you will have four weeks to complete an Final Exam - a combination of quizzes and a code project. The exam will be open from <b>July 6th, 2023</b> and you have to complete it by <b>August 4th, 2023</b>.<br/><br/>
        You will receive an email and notification via Discord closer to the date.<br/><br/>
        If you complete the program earlier you can take the exam sooner. The earliest you can take the exam is from the fourth week of the program.<br/><br/>
        The exam is an individual exercise.<br/><br/>
        <div class="tm-bold">When do I get the results?</div>
        You will receive your exam results by <span class="tm-bold">August 18th</span>.
customModules:
  - title: Weekly Plan
    description: |
      The Academy runs for seven weeks. You can follow the weekly structure or decide to go your individual path - just make sure to be ready for the Final Exam at the end of the program.
    sections:
      - image: /cosmos_dev_portal_module-02-lp.png
        title: Week 0 - Getting Started
        href: /ida-course/LPs/week-0/
        description: |
          This chapter is completely optional and a good introduction if you are new to blockchain technology or need a refresher on Golang and a short overview of dev terms you will encounter when working with Interchain Stack:
        links: 
          - title: Blockchain basics
            path: /ida-course/0-blockchain-basics/1-blockchain.html
          - title: Golang introduction
            path: /tutorials/4-golang-intro/1-install.html
          - title: Good-to-know dev terms
            path: /tutorials/1-tech-terms/
          - title: Docker introduction
            path: /tutorials/5-docker-intro/
      - image: /cosmos_dev_portal_module-03-lp.svg
        title: Week 1 - Introduction to the Interchain
        href: /ida-course/LPs/week-1/
        description: |
          You will discover the Interchain Ecosystem and learn about the main concepts of the Cosmos SDK, from its Tendermint consensus to learning how keys, accounts, and transactions relate to each other. Dive into:
        links: 
          - title: Blockchain technology and the Interchain
            path: /academy/1-what-is-cosmos/1-blockchain-and-cosmos.html
          - title: The Interchain Ecosystem
            path: /academy/1-what-is-cosmos/2-cosmos-ecosystem.html
          - title: Getting ATOM and staking it
            path: /academy/1-what-is-cosmos/3-atom-staking.html
          - title: Blockchain app architecture
            path: /academy/2-cosmos-concepts/1-architecture.html
          - title: Accounts
            path: /academy/2-cosmos-concepts/2-accounts.html
          - title: Transactions, messages, and modules
            path: /academy/2-cosmos-concepts/3-transactions.html
          - title: Protobuf
            path: /academy/2-cosmos-concepts/6-protobuf.html
          - title: Multistore and keepers
            path: /academy/2-cosmos-concepts/7-multistore-keepers.html
          - title: BaseApp
            path: /academy/2-cosmos-concepts/8-base-app.html
          - title: Queries, events, and context
            path: /academy/2-cosmos-concepts/9-queries.html
          - title: Testing
            path: /academy/2-cosmos-concepts/12-testing.html
          - title: Relaying with IBC
            path: /academy/2-cosmos-concepts/13-relayer-intro.html
          - title: Interchain Security
            path: /academy/2-cosmos-concepts/14-interchain-security.html
          - title: Bridges
            path: /academy/2-cosmos-concepts/15-bridges.html
          - title: Migrations
            path: /academy/2-cosmos-concepts/16-migrations.html
          - title: Week 1 Quiz
            path: /academy/quiz-week1.html
      - image: /cosmos_dev_portal_module-05-lp.png
        title: Week 2 - First Steps
        href: /ida-course/LPs/week-2/
        description: |
          You will discover how to run a node and learn how to build your own chain by following the example implementation of a checkers blockchain:
        links: 
          - title: Set up your work environment
            path: /tutorials/2-setup/
          - title: Run a node, API, and CLI
            path: /tutorials/3-run-node/
          - title: Introduction to Ignite CLI
            path: /hands-on-exercise/1-ignite-cli/1-ignitecli.html
          - title: Exercise - Make a Checkers Blockchain
            path: /hands-on-exercise/1-ignite-cli/2-exercise-intro.html
          - title: Store a game
            path: /hands-on-exercise/1-ignite-cli/3-stored-game.html
          - title: Create your first message
            path: /hands-on-exercise/1-ignite-cli/4-create-message.html
          - title: Create and save a game
            path: /hands-on-exercise/1-ignite-cli/5-create-handling.html
          - title: Add a way to make a move
            path: /hands-on-exercise/1-ignite-cli/6-play-game.html
          - title: Emit game events
            path: /hands-on-exercise/1-ignite-cli/7-events.html
          - title: Record the game winners
            path: /hands-on-exercise/1-ignite-cli/8-game-winner.html
          - title: Week 2 Exercise
            path: /academy/exercise-week2.html
      - image: /planet-pod.svg
        title: Week 3 - IBC and CosmJS
        href: /ida-course/LPs/week-3/
        description: |
          You will dive into IBC to learn more about cross-chain communication and take a look at how to use CosmJS for your chain:
        links: 
          - title: What is IBC?
            path: /academy/3-ibc/1-what-is-ibc.html
          - title: IBC/TAO - connections, channels, and clients (OPTIONAL)
            path: /academy/3-ibc/2-connections.html
          - title: IBC token transfer
            path: /academy/3-ibc/7-token-transfer.html
          - title: Interchain accounts (OPTIONAL)
            path: /academy/3-ibc/8-ica.html
          - title: IBC middleware (OPTIONAL)
            path: /academy/3-ibc/9-ibc-mw-intro.html
          - title: IBC tooling
            path: /academy/3-ibc/12-ibc-tooling.html
          - title: What is CosmJS?
            path: /tutorials/7-cosmjs/1-cosmjs-intro.html
          - title: Your first CosmJS actions
            path: /tutorials/7-cosmjs/2-first-steps.html
          - title: Compose complex transactions
            path: /tutorials/7-cosmjs/3-multi-msg.html
          - title: Learn to integrate Keplr
            path: /tutorials/7-cosmjs/4-with-keplr.html
          - title: Create custom CosmJS interfaces
            path: /tutorials/7-cosmjs/5-create-custom.html
      - image: /planet-collection.svg
        title: Week 4 - Ignite CLI Advanced and IBC Advanced
        href: /ida-course/LPs/week-4/
        description: |
          You will dive deeper into customizing the checkers blockchain to make your game more interesting and unique with Ignite, while also testing and expanding your IBC knowledge to:
        links: 
          - title: Keep a game deadline
            path: /hands-on-exercise/2-ignite-cli-adv/1-game-deadline.html
          - title: Keep a move count
            path: /hands-on-exercise/2-ignite-cli-adv/2-move-count.html
          - title: Put your games in order
            path: /hands-on-exercise/2-ignite-cli-adv/3-game-fifo.html
          - title: Auto-expiring games
            path: /hands-on-exercise/2-ignite-cli-adv/4-game-forfeit.html
          - title: Let players set a wager
            path: /hands-on-exercise/2-ignite-cli-adv/5-game-wager.html
          - title: Handle wager payments
            path: /hands-on-exercise/2-ignite-cli-adv/6-payment-winning.html
          - title: Integration tests
            path: /hands-on-exercise/2-ignite-cli-adv/7-integration-tests.html
          - title: Incentivize players
            path: /hands-on-exercise/2-ignite-cli-adv/8-gas-meter.html
          - title: Help find a correct move
            path: /hands-on-exercise/2-ignite-cli-adv/9-can-play.html
          - title: Play with cross-chain tokens
            path: /hands-on-exercise/2-ignite-cli-adv/10-wager-denom.html
          - title: Understand IBC denoms
            path: /tutorials/6-ibc-dev/
          - title: Go relayer
            path: /hands-on-exercise/5-ibc-adv/1-go-relayer.html
          - title: Hermes relayer
            path: /hands-on-exercise/5-ibc-adv/2-hermes-relayer.html
      - image: /cosmos_dev_portal_module-04-lp.png
        title: Week 5 - CosmJS Advanced
        href: /ida-course/LPs/week-5/
        description: |
          You will build on your previous work with CosmJS to implement a sound game GUI and a backend script that improves the user experience by:
        links: 
          - title: Create custom objects
            path: /hands-on-exercise/3-cosmjs-adv/1-cosmjs-objects.html
          - title: Create custom messages
            path: /hands-on-exercise/3-cosmjs-adv/2-cosmjs-messages.html
          - title: Get an external GUI
            path: /hands-on-exercise/3-cosmjs-adv/3-external-gui.html
          - title: Integrate CosmJS and Keplr
            path: /hands-on-exercise/3-cosmjs-adv/4-cosmjs-gui.html
          - title: Backend script for game indexing
            path: /hands-on-exercise/3-cosmjs-adv/5-server-side.html
      - image: /moving-objects.svg
        title: Week 6 - IBC Deep Dive
        href: /ida-course/LPs/week-6/
        description: |
          Ready for an IBC deep dive? In this chapter, you will further deepen your knowledge of IBC by looking into:
        links: 
          - title: IBC app developer introduction
            path: /hands-on-exercise/5-ibc-adv/3-ibc-app-intro.html
          - title: Make a module IBC-enabled
            path: /hands-on-exercise/5-ibc-adv/4-ibc-app-steps.html
          - title: Adding packet and acknowledgement data
            path: /hands-on-exercise/5-ibc-adv/5-ibc-app-packets.html
          - title: Extend the checkers game with a leaderboard
            path: /hands-on-exercise/5-ibc-adv/6-ibc-app-checkers.html
          - title: Create a leaderboard chain
            path: /hands-on-exercise/5-ibc-adv/79-ibc-app-leaderboard.html
      - image: /universe.svg
        title: Week 7 - From Code to MVP to Production and Migrations
        href: /ida-course/LPs/week-7/
        description: |
          In this chapter, you will build on your previous work for the checkers blockchain to adapt your blockchain to the demands of running in production:
        links: 
          - title: Run in Production
            path: /tutorials/9-path-to-prod/1-overview.html
          - title: Prepare the software to run
            path: /tutorials/9-path-to-prod/2-software.html
          - title: Prepare a validator and keys
            path: /tutorials/9-path-to-prod/3-keys.html
          - title: Prepare where the node starts
            path: /tutorials/9-path-to-prod/4-genesis.html
          - title: Prepare and connect to other nodes
            path: /tutorials/9-path-to-prod/5-network.html
          - title: Configure, run, and set up a service
            path: /tutorials/9-path-to-prod/6-run.html
          - title: Prepare and do migrations
            path: /tutorials/9-path-to-prod/7-migration.html
          - title: Simulate production in Docker
            path: /hands-on-exercise/4-run-in-prod/1-run-prod-docker.html
          - title: Tally player info after production
            path: /hands-on-exercise/4-run-in-prod/2-migration-info.html
          - title: Add a leaderboard as a module
            path: /hands-on-exercise/4-run-in-prod/3-add-leaderboard.html
          - title: Migrate the leaderboard module after production
            path: /tutorials/9-path-to-prod/4-migration-leaderboard.html
          - title: Simulate production migration with Docker Compose
            path: /hands-on-exercise/4-run-in-prod/5-migration-prod.html
---

This repo contains the code and content for the published [Cosmos SDK Tutorials](https://tutorials.cosmos.network/).

Note: The layout metadata at the top of the README.md file controls how the tutorial page is published. Write permissions are limited to preserve the structure and contents.

These tutorials guide you through actionable steps and walk-throughs to teach you how to use Ignite CLI and the Cosmos SDK. The Cosmos SDK is the world’s most popular framework for building application-specific blockchains. Tutorials provide step-by-step instructions to help you build foundational knowledge and learn how to use Ignite CLI and the Cosmos SDK, including:

* Foundational knowledge to help you navigate between blockchains with the Cosmos SDK
* Learn how Ignite CLI works
* Create a blockchain polling application
* Build an exchange that works with two or more blockchains
* Interact with the Cosmos Hub Testnet to test the functionality of your blockchain
* Use the liquidity module, known on the Cosmos Hub as Gravity DEX, to create liquidity pools so users can swap tokens
* Publish your blockchain application to a Droplet on DigitalOcean
* Connect to a testnet
* Design, build, and run an app as a scavenger hunt game

The code and docs for each tutorial are based on a specific version of the software. Be sure to follow the tutorial instructions to download and use the right version.

Use the tutorials landing page as your entry point to articles on [Interchain blog](https://blog.cosmos.network/), videos on [Interchain YouTube](https://www.youtube.com/c/CosmosProject/videos), and ways to get help and support.

This repo manages and publishes the tutorials. For details, see [CONTRIBUTING](/sdk-tutorials/CONTRIBUTING.md) and [TECHNICAL-SETUP](/sdk-tutorials/TECHNICAL-SETUP.md).
The tutorials are formatted using [markdownlint](https://github.com/DavidAnson/markdownlint/blob/main/doc/Rules.md).
