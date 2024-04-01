# zkVM Part Two - Design Considerations of zkVM

![alt text](pics/zkVM-PartTwoBanner.png)
*The Realm of “ZK Everything” - General-purposed Zero Knowledge Virtual Machine Blog Series.*

*By @BrianSeong99*

*Special Thanks to Keith Chen from SNZ for the review!*

This section will require extensive computer science fundamentals to understand. The most important topic would be understanding Computer Architecture, where you learn about CPU Pipelines, Instruction sets, Machine Types, Data Flow Control, storage/memory models, and also cryptography basics. (I took mine back in 2019 learning about how to implement MIPs using Verilog and never thought one day I would actually make use of the knowledge ;)


All of the design considerations are there to serve this purpose: to make a zk-friendly execution trace that can generate and verify proofs quickly and in small sizes. This being the first priority, there are other properties including safety, useability, and more, which are also gradually being fulfilled.


I will explore the design constraints of two major components of the zkVM in this article, first the Virtual Machine, and then the Zero Knowledge part.


<center>----------Enjoy!-----------</center>
<br/>
<br/>

# Virtual Machine
I will explain a little bit about what are the design factors to consider when implementing VM. It’s okay to not understand everything in this section, you will have to come back to this section when you start reading part 3 of the blog series to understand them better.


### Instruction sets
- Mainstream instructions like RISC V, WASM
- EVM instructions, this would be mostly for zkEVM
- Zero Knowledge specific instructions like Miden, Cairo

### Machine Type
- Stack Machine: this is the simple kind of machine that follows the rule of Last-In-First-Out (LIFO), where the most recently added data would be the first to be removed. 

- Register Machine: this is the more adopted kind of machine type, especially for CPU. It was created because having to access data from Cache or Memory during ALU calculation would be a daunting task, wasting a lot of time. Therefore having an immediately accessible on-cpu-storage that is really small and ultra fast to access would speed up the CPU operation exponentially.

- Memory-Memory Machine: this is exactly opposite to Register Machine where most of the data is stored on the memory, which could encourage the instruction sets to be simpler, since data read and write would only be using “MLOAD”, “MSTORE”, and no need to consider using Register Data operations.

### Flow Control
- Harvard Architecture: named after the Harvard Mark 1 computer, it is a design from the very beginning time of the modern computer. It separates its memory into instruction memory and data memory, this design makes accessing memory more efficient where it can use two pipelines to access instruction and data at the same time. There are some more efficient designs that would even divide the cache into 2 sections to parallel with memory. It’s usually used for embedded devices for more application-specific use cases for more efficient processing.

- Von Neumann Architecture: If you have taken a computer science degree, you should know about this name. This is the most widely adopted computer architecture that works among computers, servers, phones, and more. The magic is that instead of using 2 separate memory sections for instruction and data, it uses a unified single memory for data access. Due to its bigger memory size and versatility, Von Neumann Architecture is mostly used for general wide range of applications.

- Merkelized Abstract Syntax Tree: You might have seen it somewhere in other places in blockchain, in this case, it's about composting assembly codes into leaves and nodes. Here are some of the traits of MAST:
  - All programs can be reduced to a single hash (the root of the MAST)
  - Every internal node is itself a MAST of a smaller program
  - Leaves of a program MAST are linear programs (no control flow)
  
  Basically, the tree nodes are accessed from the leftmost node and go through all the nodes and leaves via the left traversal tree.


### Native Field
- 32 bits, 64 bits, 128 bits, 256 bits: the field size directly affects the performance of proof generation as most hardware is 64 bits, and there are advanced hardware acceleration instructions like avx2 that run on 32 bits. But there are also numbers that can only be housed in 256-bit format. So figuring out ways to make good use of hardware and make sure software is optimized for the hardware capability is essential for optimizing the performance of ZK proof generation. 

### Other Considerations:
- Program initialization: This is the initial setup phase of the VM program. Might involve ZK setup as well depending on the design of the ZK.

- Function calls: static or dynamic call targets, static calls are jump access of a function logic in assembly code with a static address, which means the function will always be at that location in memory/program. Dynamic calls would be more complicated and the location of the dynamic function would only be calculated when the program is executing. 

- Memory Model: Strategies to manage memory. It is self is another complex topic, in short is basically how to organize and manage memory space so that accessing data is fast, with low cost, and adaptable. There are many factors to consider depending on the application, I encourage you to do some more research on this part if you are interested. 

- Native Data types: Primitive Data types, Dynamic Data types, field elements, and more.


I got this design considerations guideline from a [talk](https://youtu.be/81UAaiIgIYA?si=p2qJMkIU7MNrTTYu&t=528) from the founder of @[0xPolygonMiden](https://twitter.com/0xpolygonmiden), @[bobbinth](https://twitter.com/bobbinth), and added my own explanations. Honestly, it is one of the best talks to quickly grasp an understanding of zkVM!


Zero Knowledge Proof
Zero Knowledge Proof, as the crown technology to scale Ethereum, is a cryptographic system that proves a statement/result without revealing the process/data. In the context of computer programs, it usually refers to showing the result of the code execution, and proof that the execution process is authentic, without revealing the actual execution process.


There are generally two ways to go about this:

Circuit Based ZKP: Building application-specific circuits of the program so that the entire execution is done on the circuit.

Virtual Machine Based ZKP: Build a Virtual Machine execution trace circuit, so that the program is executed on the virtual machine, and its execution trace is fed to the circuits as input. 

Basically, zkVM is the later implementation where it only requires the VM builder to write the circuit once, and programmers would just need to write their code in the VM’s supported programming language, and no need to worry about writing their own circuits to build zk applications. One could also understand this as making a wrapper around circuit based ZKP so that developers would just need to know how to use the wrapper. It’s not very accurate to describe in this way but it certainly would help with understanding.


Here are some of the key concepts before we talk about the proof generation procedure:

Execution Trace
Execution Trace is a complete state recording of the machine state at every clock cycle of program execution. Usually, it is in the form of a 2D table, where each row is the machine state of each clock cycle count, and each column is the entire record of the machine’s variables, constraints, flags, and more.


Arithmetization
A process of converting the execution trace table into an algebraic problem. This is extremely important as this is the preparation phase of the PIOP process to find interpolation of the execution trace and generate polynomial commitment.


Interactive Oracle Proof (IOP)
An interactive proving system where the verifier challenges the prover multiple rounds of inputs then the verifier decides whether to accept or reject if the prover is authentic. Here’s a good paper from Eli Ben-Sasson explaining IOP. There are several IOPs that are already in use in many protocols, including PCP, QAP, FRI, STARK, Bulletproofs, and more.


Polynomial Commitment Scheme (PCS)
A Commitment Scheme is basically a commitment of information and only sharing the commitment to the verifier. It is meant to keep the original information private and the shared commit can only be generated from the original information. A Polynomial Commitment Scheme is basically a commitment of a polynomial specifically, and additionally, it is able to take open challenges of certain input challenges and share the polynomial result of the inputs for evaluation. Basically trying to prove that I(Provers) have a polynomial that works without revealing it to others(Verifiers). Existing PCS includes KZG, IPA, FRI, PLONK, Marlin, Sonic, and more.


During the execution of PIOP, there are usually steps that use PCS to commit the polynomial for later prover-verifier interactions. Depending on the choice of PCS, the setup and calculation required for generating proof differs, therefore resulting in different proving generation speeds, verification speeds, proof sizes, and more. It is up to the protocol builder to decide which properties to prioritize, and it's crucial for application builders to understand the differences and choose the right one for production use.


Other key concepts
Field: Refers to the number field the zk stack uses, it is also represented in bit size, and the performance of the zkVM also heavily depends on this factor as well as the virtual machine’s Native Field size, they would be matching with one another, or at least Field size in zk Stack should be equal or smaller than the VM’s Native Field size. Some examples of number fields include Finite Fields, Prime Fields, Elliptic Curve Fields, Goldilock Fields, and more.

Group/Hash: Specific Group or Hash function used for Polynomial Commitment Scheme, depending on the choice, There are differences in setup of the proof generation, speed of generating proof, verification speeds, and more. For instance, KZG and IPA are Group, while FRI uses Hashs.

Recursion: Recursion is the process of reducing the proof size into smaller and smaller chunks in a number of steps, eventually making it small enough but also holding essential information for the verifier to verify.

ZK Framework
All of the key terms mentioned above can be all packed into a Zero Knowledge Proof framework, such as Halo2, Plonky2, and a new framework Plonky3. I won’t go too deep into this topic as it itself is another big topic. (Placeholder, I will paste my blog link here in the future)


Process of zkVM Proof Pipeline
The general pipeline of every zkVM works a little differently, here I can share a simple example(STARK-based zkVM):

After the program is done running, a full record of the execution trace is recorded

Run Encoding and Arithemetization over the execution trace, which converts the trace data into an algebraic problem.

Calculate the polynomial(proof), and commit the polynomial with the choice of PCS.

Optional, run through recursion to reduce the polynomial size.

Verifiers will challenge the prover by giving inputs/constraints. 

The prover will provide the relevant nodes and branches of the Merkel tree required for verification. 

After the verifier gets it, it will calculate and verify whether it is equal to the root of the Merkel tree provided before.

Repeat steps 5 to 7 for k rounds.

The verifier chooses to accept or reject the prover.

A side note, you might be thinking, “Wait, wouldn’t there be quite some overhead during the k rounds of challenges? There’s also other potential threats like Internet Delay, loss packet, and more?” Basically, in actual production, the Fiat-Shamir Heuristic is applied which turns interactive protocol into a protocol that can be run statically without having to have a verifier and prover dynamically interacting for rounds.


----------End!-----------


Next, I will share my insights into the existing zkVM projects and their implementations. Hopefully, this could help developers understand better which is the right zkVM for them to use for their own builds.

Thanks for reading, let me know your comments on my tweet posts.