# A1 NFC: Faster, Dynamic, More Reliable NFC Transactions

### By Mike Kelly, Overlay Payments  
**Date**: September 9th 2024
**Email**: mike@overlaypayments.com

## Abstract
A1 is a highly compact and efficient binary messaging protocol designed for NFC applications as an alternative to the traditional APDU protocols currently used in many NFC devices such as contactless credit cards. A1 is ready for adoption by mobile phone operating systems, NFC terminal devices, and smart cards.

Built on top of ISO 14443, A1 enables streamlined dynamic application negotiation between the initiating device and the responding device, completing this process in a single round trip while minimizing metadata and communication overhead.

This paper explains the core principles, message flow, and benefits of A1, demonstrating its efficiency compared to APDU and its suitability for high-performance applications like contactless payments, access control, and IoT systems. A1 allows mobile operating systems to make NFC faster and more reliable, and can be integrated today. A reference implementation is available for iOS, written in Swift ([SwiftA1](https://github.com/overlay-payments/SwiftA1)).

## Introduction

The APDU protocol, while widely used for NFC communication, often imposes unnecessary overhead in terms of metadata and round trips, which negatively impacts performance. This can be especially problematic in latency-sensitive environments such as contactless payments, access control, and IoT.

The A1 NFC protocol addresses these challenges by minimizing both the metadata required and the number of round trips necessary for application negotiation and data exchange. By using CBOR (Concise Binary Object Representation) as its foundation, A1 ensures efficient message formatting, leveraging well-established processing rules and developer tools available across many platforms. This results in faster and more reliable interactions between NFC devices.

A1’s design focuses on two main goals:
1. **Reducing metadata overhead**: By using CBOR, A1 reduces the amount of data needed for communication, resulting in faster transactions and reduced failure rates.
2. **Minimizing round trips**: A1 can complete the application negotiation and initial data exchange in a single round trip, simplifying the communication flow and improving overall speed. For some applications, the process can be completed with just an Initiation Command (IC) and an Initiation Response (IR), completing the entire NFC interaction with a single round trip.

For instance, in widely-used NFC protocols like the Proximity Payment System Environment (PPSE), the terminal cannot communicate its supported applications or preferences when interacting with a card or device. The PPSE command merely requests a list of applications, leaving the terminal to choose from whatever the card offers, often leading to inefficient or suboptimal selection. In contrast, A1 enables a dynamic negotiation between the initiating and responding devices, where both sides can communicate supported applications and preferences, optimizing the interaction from the start.

## Protocol Overview

The A1 protocol consists of several key messages that govern the interaction between the initiating device (which starts the communication) and the responding device (which reacts to it). The protocol structure is based on CBOR, ensuring that the messages are both compact and easily processable.

The main message types in A1 are:

1. **Initiation Command (IC)**: Sent by the initiating device, this message lists the Claimed IDs (CIDs) and includes any application-specific data in a CBOR map format.
2. **Initiation Response (IR)**: Sent by the responding device, this message confirms the supported CIDs and includes any application-specific data, also in a CBOR map.
3. **Kickoff Command (KC)**: Sent by the initiating device, this is a CBOR sequence that identifies the chosen CID and includes any application-specific data for that CID. This message is optional.
4. **Kickoff Response (KR)**: Sent by the responding device, this message contains arbitrary bytes. This message is also optional.

Some applications may complete the interaction with just the IC and IR, making the Kickoff Command (KC) and Kickoff Response (KR) unnecessary.

Unlike PPSE, where the terminal must passively accept whatever application the card offers, A1 allows the initiating device to directly communicate its supported applications and preferences in the Initiation Command (IC). This leads to a more dynamic negotiation process between the devices, reducing the need for multiple messages and enabling faster, more precise application selection.

### Message Flow

The interaction between the initiating and responding devices is designed to be efficient, with the key negotiation and data exchange typically completed within the first two messages. The **Kickoff Command (KC)** and **Kickoff Response (KR)** are optional, and some applications may finish the interaction with only the IC and IR.

```plaintext
Initiating Device                      Responding Device
 |                                        |
 |  Initiation Command (IC)               |
 |--------------------------------------->|
 |                                        |
 |  Initiation Response (IR)              |
 |<---------------------------------------|
 |                                        |
 |  (Optional) Kickoff Command (KC)       |
 |--------------------------------------->|
 |                                        |
 |  (Optional) Kickoff Response (KR)      |
 |<---------------------------------------|
```

## Key Design Principles

### 1. Minimized Protocol Metadata

A1 minimizes the amount of protocol metadata by leveraging CBOR, a highly efficient data format. CBOR’s compactness ensures that A1 consumes fewer bytes for metadata, allowing more efficient use of NFC’s limited bandwidth. This is especially important in environments where speed and performance are critical. Additionally, CBOR is well-supported across various platforms, meaning developers can easily adopt A1 without needing specialized tools or frameworks.

A1 establishes a new registry for application identifiers called Claimed IDs which can be single bytes, further reducing protocol overhead.

### 2. Reduced Round Trips

A1 minimizes the number of round trips required to complete an NFC interaction by combining application negotiation and data exchange into just one command and response. This reduction in communication is crucial for improving both efficiency and reliability, particularly in high-demand environments such as contactless payments and access control.

#### Key Benefits of Fewer Round Trips

1. **Less Protocol Overhead**:  
   In ISO 14443 communication, each command and response comes with protocol overhead, which consists of several components necessary for ensuring correct data transmission and reception. These include:

   - **Preamble and Start Frame Delimiter**: Used to signal the start of a command or response message.
   - **Command/Response Header**: Includes fields for the instruction, parameters, and status.
   - **Error Checking Code**: Used to detect transmission errors, which requires additional bytes of data for every exchange.
   - **Response Time-out Handling**: ISO 14443 introduces a time-out mechanism to handle lost or delayed responses, which adds further complexity to each communication round trip.

   In a traditional ISO 14443-based interaction using APDU, multiple round trips are often required just for application selection and data exchange, with each round trip carrying the above overhead components. This accumulates, consuming bandwidth and prolonging the interaction.

   A1 addresses this by reducing the number of round trips, which cuts down on the repeated inclusion of this overhead. By consolidating negotiation and data exchange into fewer messages, A1 streamlines communication, ensuring that the majority of the interaction consists of useful data rather than protocol management details. This leads to faster transactions and less errors.

2. **Resilience**:  
   Each additional round trip introduces more opportunities for errors or interruptions—whether due to signal interference, processing delays, or dropped packets. By reducing the number of communication exchanges, A1 decreases the likelihood of these disruptions. Fewer round trips mean fewer points of failure, resulting in more reliable and smoother interactions, which is critical in applications like payments or access control where any delay or failure could impact the user experience.

A1’s ability to combine application negotiation and data exchange within the **Initiation Command (IC)** and **Initiation Response (IR)** minimizes the need for additional messages. In many cases, the interaction can be completed with just these two messages, while the **Kickoff Command (KC)** and **Kickoff Response (KR)** are available for more complex applications if needed.

### 3. Error Handling

A1 simplifies error handling by treating any non-conforming Initiation Response (IR) as an error. Implementers may inspect the response to determine the nature of the failure. Furthermore, any APDU message received during an A1 interaction is treated as an error, ensuring A1 is interoperable with existing ISO 7816 tags and devices.

## Message Types and Examples

### 1. Initiation Command (IC)

The **Initiation Command** is the first message sent by the initiating device, which begins the negotiation process. It includes:

- A CBOR map where the keys represent CIDs, indicating the applications that the initiating device supports.
- The values of the map are CBOR byte strings containing application-specific data. If no data is necessary, a zero-length byte string is used.

#### Example (CBOR format):
```plaintext
{
  0x0001 => '',
  0x0002 => 0x494c4f56454131
}
```

### 2. Initiation Response (IR)

The **Initiation Response** is sent by the responding device after receiving the IC. It confirms the CIDs that the responding device supports and may contain application-specific data as well. Like the IC, the IR uses a CBOR map structure.

#### Example (CBOR format):
```plaintext
{
  0x0002 => 0x49414c534f3c334131
}
```

### 3. Kickoff Command (KC)

Once the initiating device receives the IR, it may send a **Kickoff Command** to finalize the application selection. The KC uses a CBOR sequence, consisting of:

- The selected CID, explicitly identifying the chosen application.
- A CBOR byte string containing application-specific data, if needed.

The KC is optional and only used if further communication is required.

#### Example (CBOR format):
```plaintext
[
  0x0002,
  0x4a55494345544841545448494e47544f524544
]
```

### 4. Kickoff Response (KR)

The **Kickoff Response** is the final message in this process, sent by the responding device. It contains arbitrary bytes, providing flexibility for the application to define the format and content of the response without adhering to any strict data format requirements such as CBOR. Like the KC, the KR is optional.

#### Example (Arbitrary format):
```plaintext
0x5448494e474a5549434544544f524544
```

## Example A1 Session

```
Initiation Command
--------------------------
raw: 0xa1416945416c696365
CID map:
{
  0x69 : 0x416c696365
}

Initiation Response
--------------------------
raw: 0xa1416943426f62
CID map:
{
  0x69 : 0x426f62
}


Kickoff Command
--------------------------
raw: 0x416947596f20426f6221
CID: 0x69
Application Data: 0x596f20426f6221


Kickoff Response
--------------------------
raw: 0x486920416c69636521
```

Here is what is happening in the above example:
- Alice and Bob are using an application with the Claimed ID 0x69
- They share initiation messages to negotiate and agree to speak 0x69
- The 0x69 application allows attaching names to the initiation messages encoded as utf8 strings
- Alice knows Bob's name from the Initiation Response, so can greet him in the Kickoff Command.
- Bob knows Alice's name from the Initiation Command, so can greet him in the Kickoff Response.

## Integration with Mobile Operating Systems

To ensure that applications built on A1 can work seamlessly on mobile platforms, mobile operating system vendors can integrate A1 into their NFC stack today, enabling applications to declare the CIDs they support or intend to initiate.

- **Host Card Emulation (HCE)**: Applications that act as the responding device in HCE mode should declare the CIDs they support up front, typically in the application manifest. When an NFC interaction occurs, the operating system will route messages based on the declared CIDs, ensuring that the correct application handles the communication. In transit or other use cases where the HCE application has not been pre-selected by the user, the operating system must manage the priority of applications for a given CID to prevent conflicts.
  
- **Reader Mode**: In reader mode, where the application acts as the initiating device, the application should declare the CIDs it intends to initiate in its manifest. The operating system should enforce this declaration, ensuring that the app only uses these CIDs in its Initiation Command (IC) and Kickoff Command (KC). This ensures secure and predictable behavior across applications.

A reference implementation for iOS is available in Swift ([SwiftA1](https://github.com/overlay-payments/SwiftA1)), demonstrating the processing rules for A1 and how the flow works.

## Use Cases

A1 is well-suited for a variety of NFC applications, particularly those that require low latency and minimal communication overhead:

- **Contactless Payments**: A1 reduces transaction times by completing negotiation and data exchange in a single round trip, leading to faster, more efficient payments.
- **Access Control**: The reduced round trips and metadata ensure quick interactions in access control systems, improving the flow of people or goods in high-traffic areas.
- **IoT Systems**: Many IoT devices benefit from A1’s efficient messaging structure, which conserves bandwidth and battery life while still delivering high-speed communication.

### Recommendation on the Use of Initiation Data

When building NFC applications on top of A1, it's important to treat the inclusion of data in the Initiation Command (IC) and Initiation Response (IR) as an **optional optimization**, rather than a strict requirement. While embedding application-specific data in these messages can reduce the need for additional communication steps, there are cases where omitting this data might be beneficial—especially for devices supporting multiple applications simultaneously.

For initiating devices, such as mobile phones or readers, managing multiple applications means that including data for every supported application in the IC could lead to unnecessary performance overhead. Instead, developers should consider only optimizing key applications by including data for a preferred subset in the IC. This allows for efficient communication without the need to overload the IC with data for every possible application, thereby improving overall performance.

Similarly, responding devices can take the same approach with the IR. Not all supported applications may need to send data upfront, and by limiting the data to key applications, the IR remains lightweight and efficient.

By designing your application so that the inclusion of data in the IC and IR is optional, you allow flexibility for devices to optimize communication based on their performance needs. This approach maintains the protocol's efficiency while ensuring that devices with multiple application requirements aren't burdened by unnecessary data transmission.

## Conclusion

The A1 NFC protocol offers a compact, efficient, and dynamic alternative to APDU. By using CBOR to minimize protocol metadata and reducing the number of round trips needed for application negotiation, A1 significantly improves performance in NFC environments. The optional Kickoff Command (KC) and Kickoff Response (KR) provide flexibility for applications that require additional messaging, but many use cases can complete with just the Initiation Command (IC) and Initiation Response (IR). The Kickoff Response (KR), containing arbitrary bytes, allows full freedom for application-specific data handling.

As NFC technology continues to evolve, A1 provides a scalable, modern solution that meets the demands of high-performance, real-time communication, particularly in use cases where efficiency and low latency are essential. Integration with mobile operating systems is possible today, making A1 a practical choice for current NFC ecosystems.

## A1 CID Registry

The A1 CID Registry is a public repository where implementers can claim their own unique Claimed IDs (CIDs) for use with the A1 NFC protocol. Each CID is a unique byte sequence that represents a specific application or service. This registry helps ensure that there are no conflicts between applications using A1.

Implementers are encouraged to contribute by raising a pull request on the [A1 GitHub repository](https://github.com/overlay-payments/A1) to claim their own CID. By doing so, they can ensure their applications are properly registered and avoid any CID clashes.

Below is a list of CIDs currently registered:

| Bytes | Description                   | URL                                                 |
|-------|-------------------------------|-----------------------------------------------------|
| 0x00  | Overlay TapToAuth v1.0.0      | [https://docs.overlaypayments.com/TapToAuth/1.0.0](https://docs.overlaypayments.com/TapToAuth/1.0.0) |
| 0x01  | Overlay TapToAuth v2.0.0      | [https://docs.overlaypayments.com/TapToAuth/2.0.0](https://docs.overlaypayments.com/TapToAuth/2.0.0) |

To register your own CID, simply submit a pull request adding your CID, description, and documentation URL to this list.
