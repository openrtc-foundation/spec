# OpenRTC Application-Facing Interoperability Specification

## 1. Abstract

This document specifies an application-facing contract for real-time communications: conference participation, observable state, local media, errors, ownership, and optional profiles. It defines observable behavior, not a signaling or media transport protocol.

## 2. Status and Conventions

This document is an OpenRTC Foundation specification. It is not an IETF publication.

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **NOT RECOMMENDED**, **MAY**, and **OPTIONAL** in this document are to be interpreted as described in BCP 14 [RFC2119] [RFC8174] when, and only when, they appear in all capitals, as shown here.

Sections 1–15 and Appendix A are normative except text labeled “Rationale”, “Example”, or “Note”. Section 16 contains normative and informative references as labeled; Appendix B is informative. Appendix A specifies operation and type shapes and complements the body; its failure-code lists are informative examples, and Section 9 governs failures. A binding MUST supply equivalent operations and representations. Names and calling syntax need not be identical.

## 3. Introduction

Applications need consistent answers to whether they joined, what they are sending, who is present, and what remains to be released. Provider APIs differ in these observable semantics. OpenRTC specifies a contract at that boundary so applications can make those decisions without assuming one provider's behavior. WebRTC supplies relevant media and connectivity technologies; SIP specifies signaling for sessions and calls. Neither is replaced or required as a wire protocol by this document.

## 4. Terminology

An **application** invokes the contract. An **implementation** realizes it using a **provider**, which may comprise a service and native software. A **trusted server** holds provider management secrets and performs privileged administration. A **client** owns local sessions; a **session** is one local conference participation, not necessarily the remote conference itself. A **participant** is one endpoint presence in that conference, not necessarily a person. A **source** produces local media; a **publication** represents sending a source. A **subscription** is a manual selection of a remote publication. An **owner** has authority over a resource's lifetime. A **binding** expresses this contract in a particular environment.

An **established** fact has supporting observation or authoritative evidence. **Unknown** means not established; **absent** means established not present; **not applicable** means the question has no meaning in that scope. These meanings MUST be distinguishable where they affect behavior. Lists describing observed populations MUST state their coverage and completeness as complete, incomplete, or unknown; unknown MUST NOT be treated as complete. Request lists and finite result lists whose contents are defined by the operation do not require population coverage. When a fact is known to be stale or verification is lost, it MUST be labeled last-known or unknown, with its observation time or validity boundary when known. A **dependent effect** is an effect whose permission or correctness depends on a requested condition. A **final** result is one already settled to the application.

## 5. Scope and Non-Goals

The core covers local conference participation, scoped identity, observation, optional audio/video controls, discovery, errors, and resource lifetime. Support for an individual media operation is conditional. Its specified behavior applies whenever supported. Optional profiles extend this contract. Internal architecture, threading, schedulers, native-reference mapping, cleanup registries, transport protocols, and specific language bindings are outside scope. A binding may provide rendering; its observable behavior is constrained by Section 8.7.

## 6. Architecture and Roles

The application obtains credentials for the intended provider and destination from an authorized source.

A binding MUST document the authority, subjects, decision and evidence source for protected actions. Unknown required authority or consent MUST fail before protected effects. Unknown credential claims do not establish a required scope match and MUST fail when the caller requires that scope. The implementation MUST use them only within their known audience, subject, purpose and destination. A known mismatch MUST fail before submission. The trusted server holds signing and management secrets and controls privileged operations. Client credentials MUST NOT confer trusted-server authority. Provider authorization remains distinct from support discovery and application intent.

Resource identifiers are opaque and identify one incarnation in a documented scope. They MUST NOT be reused for another incarnation within that scope. A stale, foreign, terminal, or unauthorized reference MUST NOT select or affect another resource, including in a later session. Source use by another owner requires the source owner's explicit grant. Sharing does not transfer ownership or release authority.

## 7. Core Operations

### 7.0 Operation requirements

The following table is normative. “Entry” requires a binding-equivalent callable entry point in the stated claim; “unsupported” refers to an otherwise valid, authorized request in the claimed scope. Errors for individual invalid requests and unavailable services remain possible. Profile names denote version 1. Operations not listed as mandatory for a profile are separately discoverable; a core entry remains required where applicable.

| Operation | Entry required? | May return `unsupported` for every valid request in claim scope? | Mandatory positive behavior for claim |
|---|---|---|---|
| createClient, join, leave, dispose, capabilities, observe, snapshot, unsubscribe | Core: yes | No | Core |
| capture, publish, setMuted, setDisabled, unpublish, stop | Core: yes | Yes | None; each supported operation obeys §7.4 |
| setSending | No; yes if advertised | Yes | Only when advertised |
| dial, accept, reject, hangup, setHold | Calling: yes | No | Calling; incoming offers must be observable |
| startReceiving, stopReceiving | Only if receiving registration advertised | Yes otherwise | Calling receiving-registration capability |
| transfer, sendDigits | Only if corresponding capability advertised | Yes otherwise | Corresponding Calling capability |
| call-scoped capture or use of granted source, call-scoped publish | If call media advertised | Yes otherwise | Calling media capability; at least one acquisition/use path |
| subscribe, unsubscribeMedia | Manual Subscription: yes | No | Manual Subscription |
| shareSource, revokeSourceGrant, replaceSource | Source Replacement and Sharing: yes | No | Source Replacement and Sharing |
| captureScreen | Screen Sharing: yes | No | Screen Sharing |
| sendMessage, observeMessages | Data Messaging: yes | No | Data Messaging |
| startRecording, stopRecording | Recording: yes | No | Recording session-scoped |
| startIndependentRecording | Only if independent job advertised | Yes otherwise | Recording independent-job capability |
| assessPlacement, applyPlacement | Placement: yes | No | Placement |
| configureEncryption | End-to-End Encryption: yes | No | End-to-End Encryption |
| createConference, endConference, removeParticipant, mintCredential | Server Administration: yes | No | Server Administration |
| listDevices, selectDevice, getStatistics | Devices and Statistics: yes | No | Devices and Statistics |
| renewCredential | Credential Renewal: yes | No | Credential Renewal |
| observeProfile | Each profile requiring observations: yes | No | Each such profile; a shared entry point is permitted |

A core implementation MUST successfully create a client, admit an authorized endpoint to a supported destination, provide capabilities, obtain a snapshot and a gap-free observation, and leave and dispose within the declared bounds under valid supported conditions. A core claim may support no media operation. Each mandatory profile operation MUST have a passing positive assertion in its claimed scope; a claim cannot pass by returning `unsupported` for every valid request. Table requirements do not grant authority.

### 7.1 Common operation contract

Each operation MUST identify its target and owner. Before effects, it MUST validate determinable input, target, current support, authority, consent, and required conditions. Invalid or unsupported requests and known authorization failures MUST fail before attributable effects. Other failures may arise after execution starts. Section 9 defines codes and detailed outcomes. No operation is limited to the codes listed in its table below.

| Code | Condition applicable to any operation |
|---|---|
| `invalid-argument` | Input is malformed, inconsistent, misplaced or invalid. |
| `invalid-state` | The target or operation is ineligible in its current state. |
| `unsupported` | The operation, requested combination or required condition cannot be provided. |
| `permission-denied` | Required authority or consent is denied. |
| `authentication-failed` | Credential or identity authentication fails. |
| `cancelled` | Accepted operation cancellation wins, or an explicitly identified user dismissal occurs. |
| `timeout` | An applicable deadline expires. |
| `unavailable` | Network or service is unavailable. |
| `internal` | An implementation failure is established. |
| `unknown` | No more specific portable condition is established. |

Each operation MUST settle exactly once. Cancellation, deadlines and late completions follow Section 8.5. Failure and cancellation outcomes follow Section 9. Known surviving application-controllable resources MUST be usable for cleanup or released. Unresolved release follows Section 8.6. Established public changes MUST be observable under Section 8.3. Success MUST NOT precede visibility of the public state needed for its promised result. An invalid indivisible update MUST NOT be partially visible. An invalid indivisible update MUST NOT cause an unrelated operation to fail solely because of that update. Overlapping operations follow Section 8.4.

An additional effect is an application-observable change to a separately identifiable resource outside the operation's specified postconditions. Before causing an additional effect, an operation MUST identify the affected resources or an explicitly bounded set. For each effect, it MUST identify the action and whether it is guaranteed, possible, or unknown. Each guaranteed or possible additional effect requires explicit acceptance before it occurs. Unknown scope or effect MUST NOT satisfy required isolation. If absence of unaccepted additional effects within the implementation's observable authority cannot be established, the operation MUST reject `unsupported` before effects. Ending dependent local publications is a named postcondition of stopping a source.

Before creating a resource, the implementation MUST disclose known inseparable effects of its later release and obtain explicit acceptance. Owner-authorized release MUST proceed without new acceptance for effects already accepted. If an undisclosed inseparable effect prevents safe release, the implementation MUST report the affected scope and unresolved cleanup and MUST NOT claim release. These rules also apply to compound operations and late-acquired resources.

The failure tables in Sections 7.2–7.4 identify important operation-specific conditions. The common table and Section 9 also apply. Events are established changes delivered to observers under Section 8.3; an operation need not emit an event when no public state changes. Cancellation and late completion of each asynchronous operation follow Section 8.5 unless its block states a more specific obligation.

### 7.2 Client and session

**createClient(options)**

- **Purpose:** Create a client.
- **Inputs:** Initialization options under Section 8.8.
- **Preconditions:** Options and required conditions are valid and supportable.
- **Postconditions on success:** Return a ready client.
- **Failures:**

  | Code | Condition |
  |---|---|
  | `invalid-argument` | Initialization option is invalid or misplaced. |
  | `unsupported` | A required option or combination cannot be provided. |

  Failure MUST return no usable client. Acquired resources MUST be released or unconfirmed release reported.
- **Events emitted:** No session change is implied.
- **Ownership:** The client owns its sessions. Disposal follows Section 8.6.
- **Cancellation and late completion:** Late-acquired resources MUST be released or their unconfirmed release reported.

**join(client, destination, options)**

- **Purpose:** Admit the local endpoint to the specified conference.
- **Inputs:** Ready client, identified destination, credentials, and join options.
- **Preconditions:** Credentials are valid for their known scope. Admission authority and applicable consent are required.
- **Postconditions on success:** Return a connected session owned by the client. The local endpoint MUST have been admitted. A snapshot MUST be immediately obtainable and include all locally established facts within its coverage under Section 8.3. Success does not establish media readiness. The result MUST identify each independently controllable created resource.
- **Failures:**

  | Code | Condition |
  |---|---|
  | `invalid-argument` | Destination or option is invalid. |
  | `invalid-state` | Client or target handle is ineligible. |
  | `authentication-failed` | Credential authentication fails. |
  | `permission-denied` | Admission or required consent is denied. |
  | `unsupported` | A required admission, provisioning or isolation condition cannot be met. |

  A held request MUST remain pending and MUST NOT be reported as admitted. Denial MUST fail without local participation.
- **Events emitted:** Established admission and termination are observable under Sections 8.1 and 8.3. If a pre-admission session handle is exposed, its observers receive an established terminal transition on failure; otherwise the join error carries the established reason and no session event is required.
- **Ownership:** The client owns the session. Ownership of explicitly requested media follows the relevant media operation.
- **Cancellation and late completion:** A winning cancellation fences admission and requires cleanup under Sections 8.5–8.6. A late admission MUST NOT resurrect the session.

Join alone MUST NOT capture or publish. Explicitly requested bundled capture or publication is permitted only with the separate consent and additional-effect rules of Sections 7.1 and 12. If join may create the destination, it MUST disclose that possibility or unknown before joining. An existing-only request MUST be enforced atomically or fail before joining. Automatic creation retry requires an idempotency guarantee. Unconfirmed rollback is not complete. If a binding separately exposes local opening, it has no provisioning, resolution, credential, join, capture or publish effect. Conflicting joins on that handle cannot both succeed.

**leave(session)**

- **Purpose:** End local participation.
- **Inputs:** A session.
- **Preconditions:** Release authority exists. A terminal session remains eligible for repeated leave.
- **Postconditions on success:** Local participation is terminal. Release of session-owned resources is initiated. Leave MUST NOT request termination for all participants. Local completion does not prove remote release.
- **Failures:**

  | Code | Condition |
  |---|---|
  | `permission-denied` | The caller lacks applicable release authority. |
  | `internal` | The local terminal postcondition cannot be established because of an implementation failure. |

  Unresolved cleanup MUST be reported under Section 8.6.
- **Events emitted:** The terminal state and reason are observable when established.
- **Ownership:** Release is limited to session-owned resources and authorized effects under Sections 7.1 and 8.6.
- **Cancellation and late completion:** Leave fences pending work and settles within the finite bound. Repeated leave is safe. Late completion MUST NOT restore participation.

**dispose(client)**

- **Purpose:** Close a client and initiate release of its owned resources.
- **Inputs:** A client.
- **Preconditions:** Release authority exists. A disposed client remains eligible for repeated disposal.
- **Postconditions on success:** New work is prevented. Owned sessions are terminated. Owned resource release is initiated. Local completion does not prove remote release.
- **Failures:**

  | Code | Condition |
  |---|---|
  | `permission-denied` | The caller lacks applicable release authority. |
  | `internal` | The local disposal postcondition cannot be established because of an implementation failure. |

  Unresolved cleanup MUST be reported under Section 8.6.
- **Events emitted:** Established session termination and reason are observable.
- **Ownership:** Disposal affects only resources the client is authorized to release.
- **Cancellation and late completion:** Disposal fences pending work and settles within the finite bound. Repeated disposal is safe. Late completion MUST NOT restore disposed state or terminated participation.

### 7.3 Discovery and observation

**capabilities(scope, request)**

- **Purpose:** Assess support without performing the requested operation.
- **Inputs:** Scope and requested operation or combination, including media kind where applicable.
- **Preconditions:** The scope and request are valid.
- **Postconditions on success:** Report operation, kind where applicable, resource scope, `supported`, `unsupported`, or `unknown`, known constraints, limits, coupled effects and profile versions. Conditional support MUST identify its conditions. An unevaluable condition is unknown. A requested combination MUST be assessed as a whole. Support for individual items does not establish support for their combination. The result MUST identify how changes become observable. Support is advisory and does not confer authority. Each invocation of the assessed operation MUST recheck current support and authority. Unknown or unsupported required properties MUST reject before dependent effects. A pending condition MUST identify its resolution mechanism and evaluation boundary. Pending MUST NOT mask a known denial or failure.
- **Failures:**

  | Code | Condition |
  |---|---|
  | `invalid-argument` | Request or scope is malformed. |
  | `invalid-state` | Scope is ineligible for assessment. |

- **Events emitted:** Known capability changes are observable through Section 8.3 or another documented notification.
- **Ownership:** No protected effect or consent prompt is permitted. No media is acquired.
- **Cancellation and late completion:** Section 8.5 applies to asynchronous assessments.

**observe(session)**

- **Purpose:** Observe a coherent starting view and subsequent changes.
- **Inputs:** Session and observation scope if supported.
- **Preconditions:** The caller is authorized to observe the requested scope.
- **Postconditions on success:** Establish the snapshot and stream specified in Section 8.3.
- **Failures:**

  | Code | Condition |
  |---|---|
  | `invalid-state` | Session is ineligible for observation. |
  | `permission-denied` | Observation is unauthorized. |

- **Events emitted:** Deliver the initial snapshot and subsequent covered changes under Section 8.3.
- **Ownership:** The observer is independently removable. Observation does not acquire media.
- **Cancellation and late completion:** Section 8.5 applies to asynchronous registration. A cancelled registration MUST NOT subsequently deliver events.

**snapshot(session)**

- **Purpose:** Read a local view.
- **Inputs:** Session and observation scope if supported.
- **Preconditions:** The caller is authorized to read the requested scope.
- **Postconditions on success:** Return a detached coherent local view under Section 8.3.
- **Failures:**

  | Code | Condition |
  |---|---|
  | `invalid-state` | Session is ineligible for the requested snapshot. |
  | `permission-denied` | Reading the scope is unauthorized. |

- **Events emitted:** None required.
- **Ownership:** Reading does not acquire media.
- **Cancellation and late completion:** Section 8.5 applies if the binding makes the read asynchronous.

**unsubscribe(observer)**

- **Purpose:** Stop one observation stream.
- **Inputs:** Observer.
- **Preconditions:** The observer is identifiable and the caller may remove it. Repeated removal is eligible.
- **Postconditions on success:** No further delivery to that observer occurs after completion. Removal is idempotent.
- **Failures:**

  | Code | Condition |
  |---|---|
  | `permission-denied` | The caller cannot remove the observer. |
  | `invalid-argument` | The observer reference is invalid or foreign. |

- **Events emitted:** None required.
- **Ownership:** Only the selected observer is affected. No media is acquired.
- **Cancellation and late completion:** Section 8.5 applies if removal is asynchronous.

### 7.4 Media operations

The following operations are independently discoverable by kind and scope. Each requires a live authorized target and applicable consent, except that repeated release remains available as specified in Section 8.6. No success implies remote delivery, receipt, playback, or confirmed native release. Audio and video are core media kinds. Screen capture is specified by Section 11.4.

The table applies to the named target and its dependent publications.

While disabled, captured source content MUST NOT be produced for any dependent publication. The device MAY remain open and generated silence or black frames MAY be sent. Other observable effects require Section 7.1 disclosure. Enable succeeds only when required consent and acquisition are established and the source can produce when requested; it need not prove frame delivery and MUST NOT create a publication. “Suppressed” means source-derived outbound content is suppressed; it does not mean no packets are sent. Other restrictions remain effective in every row. Terminal postconditions for unpublish and stop are local: the local publication ends and the local source becomes unusable, respectively. Native or remote release is separately observable under Section 8.6. A request that cannot establish its stated isolation or local terminal postcondition MUST reject `unsupported` before effects.

| Operation | Current state or intent | Result on success |
|---|---|---|
| capture | Authorized session; requested kind supported | A new live session-owned source is returned; no publication or sending is caused. |
| publish | Live authorized source, enabled or disabled | A new publication of that source is returned; source disable intent and existing publication mute intents remain unchanged. A disabled source remains suppressed. |
| setMuted(true) | Active publication, source enabled or disabled | Target mute intent is set and its source-derived outbound content is suppressed; source and publication remain. |
| setMuted(false) | Active publication, source enabled | The setMuted intent is cleared; the setSending intent and other restrictions remain. |
| setMuted(false) | Active publication, source disabled | The setMuted intent is cleared; the setSending intent remains, and disable continues to suppress source-derived outbound content. |
| setDisabled(true) | Live source, with zero or more active publications | Source production is suspended and every dependent publication is suppressed; their mute intents and identities remain. |
| setDisabled(false) | Live disabled source, including muted publications | Disable intent is cleared only when production is eligible; each muted publication remains suppressed. |
| setDisabled(false) | Stopped source | No success; reject `invalid-state`. |
| unpublish | Active publication, muted or unmuted; source enabled or disabled | Selected publication ends; source remains usable with its disable intent and capture resource retained; production remains suspended while disabled. |
| stop | Live source, enabled or disabled, with zero or more publications | Source becomes unusable and its dependent local publications end. |
| unpublish or stop | Corresponding target already ended or stopped | Repeated release succeeds without reacquisition or resurrection, subject to identity and authority checks. |
| setSending(false) | Active publication, regardless of mute or disable | Set and record this convenience’s publication mute intent; source-derived content is suppressed without acquiring capture. |
| setSending(true) | Active publication, regardless of mute or disable | Clear only the intent set by setSending(false); other mute and disable intents and restrictions remain effective. |
| Any setter | Target terminal | Reject `invalid-state`; do not resurrect the target. |

**capture(session, kind, options)**

- **Purpose:** Acquire local media without publishing it.
- **Inputs:** Session, requested kind and capture options.
- **Preconditions:** Session and requested scope are eligible. The caller has authority and applicable device consent. Isolated capture is supported.
- **Postconditions on success:** Return a live session-owned source of the requested kind. Capture MUST NOT join, publish or send.
- **Failures:**

  | Code | Condition |
  |---|---|
  | `invalid-argument` | Kind or capture options are invalid. |
  | `invalid-state` | Session is ineligible. |
  | `permission-denied` | Device permission or required consent is denied. |
  | `unsupported` | Kind, requested options or isolated capture cannot be provided. |
  | `unavailable` | Required service is unavailable; a device acquisition failure is reported with the applicable Section 9 code and reason. |

- **Events emitted:** Established source creation and state changes are observable under Section 8.3. A snapshot whose declared coverage includes local sources MUST include the observed local source population and its established state, including sources not published, with coverage and completeness under Section 4.
- **Ownership:** The session owns the acquired source.
- **Cancellation and late completion:** A source acquired after winning cancellation MUST be released or its unresolved release reported under Sections 8.5–8.6.

**publish(session, source)**

- **Purpose:** Create an outbound publication of a source.
- **Inputs:** Session and source.
- **Preconditions:** The source is live and authorized for the session. Publication of the requested scope is supported. Applicable consent is established.
- **Postconditions on success:** Create and return the requested publication of the specified source. Identify every independently controllable publication created. Publish MUST NOT acquire a source. Publish MUST NOT change existing mute or disable intent.
- **Failures:**

  | Code | Condition |
  |---|---|
  | `invalid-state` | Source or session is ineligible. |
  | `permission-denied` | Source use or publication is unauthorized. |
  | `unsupported` | Requested publication or isolation cannot be provided. |

  Partial creation MUST be reported under Section 9. Known surviving controllable publications MUST be usable for cleanup or released.
- **Events emitted:** Established publication creation and changes are observable under Section 8.3.
- **Ownership:** The session owns its publication. Publication does not transfer ownership of an externally owned source.
- **Cancellation and late completion:** Winning cancellation fences success. Late-created publications MUST be ended or unresolved release reported.

**setMuted(publication, muted)**

- **Purpose:** Set publication-scoped mute intent.
- **Inputs:** Active publication and Boolean mute intent.
- **Preconditions:** The caller may control the publication. The requested isolated effect is supported.
- **Postconditions on success:** When true, suppress source-derived outbound content on the selected publication while retaining the publication and source. Silence, black frames or absence may be sent. Success for true requires established current suppression. When false, clear the setMuted mute intent only when current policy and authority permit. Clearing it MUST NOT clear the setSending mute intent, source disable or another restriction. Mute does not promise zero packets, device release, or recording exclusion.
- **Failures:**

  | Code | Condition |
  |---|---|
  | `invalid-state` | Publication is ended or otherwise ineligible. |
  | `permission-denied` | Control or required consent is denied. |
  | `unsupported` | Targeted suppression or retained source and publication cannot be established. |

- **Events emitted:** Established mute intent and suppression changes are observable under Section 8.3.
- **Ownership:** Only the authorized publication is targeted; additional effects require Section 7.1 acceptance.
- **Cancellation and late completion:** Section 8.5 applies. Acknowledgment or stale state alone MUST NOT establish current suppression.

**setDisabled(source, disabled)**

- **Purpose:** Set source-scoped disable intent.
- **Inputs:** Live source and Boolean disable intent.
- **Preconditions:** The caller has authority over the source and affected scope. The requested effect is supported.
- **Postconditions on success:** When true, suspend source-derived production and suppress it on all dependent publications. Those publications MUST NOT be represented as unpublished solely because of disable. When false, clear disable intent only when production is eligible, including any required reacquisition. Clearing disable MUST NOT clear mute or another restriction. A stopped source MUST NOT be enabled.
- **Failures:**

  | Code | Condition |
  |---|---|
  | `invalid-state` | Source is stopped or otherwise ineligible. |
  | `permission-denied` | Source control, affected scope or required consent is denied. |
  | `unsupported` | Required suppression or isolation cannot be established. |

  Failed reacquisition MUST NOT be reported as successful enable. Its known and unknown effects MUST be reported under Section 9.
- **Events emitted:** Established source intent, production eligibility and affected publication suppression changes are observable under Section 8.3.
- **Ownership:** Control requires authority over the source and affected scope. Disable does not transfer release authority.
- **Cancellation and late completion:** Section 8.5 applies. Late acquisitions MUST be released or unresolved release reported.

**unpublish(publication)**

- **Purpose:** End one local publication while retaining its source.
- **Inputs:** Publication.
- **Preconditions:** The caller has authority to end the publication. Repeated unpublish of the same ended publication is eligible.
- **Postconditions on success:** End the selected local publication. Leave the source usable and retain its capture resource; a disabled source remains suspended. Mute MUST NOT be reported as publication removal. Remote or native deletion is not asserted without evidence.
- **Failures:**

  | Code | Condition |
  |---|---|
  | `permission-denied` | Release authority is absent. |
  | `unsupported` | Publication termination while retaining the source cannot be established. |
  | `invalid-argument` | Reference is invalid or foreign. |

- **Events emitted:** Established publication termination is observable under Section 8.3.
- **Ownership:** The session owns the publication; unpublish does not release its source.
- **Cancellation and late completion:** Section 8.5 applies. Unresolved release remains observable under Section 8.6.

**stop(source)**

- **Purpose:** Terminate a source and its dependent local publications.
- **Inputs:** Source.
- **Preconditions:** The caller has the source owner's release authority. Repeated stop of the same stopped source is eligible.
- **Postconditions on success:** Make the source unusable. Prevent further source-derived production. End all dependent local publications. Initiate release of the source's capture resources. Remote or native deletion is not asserted without evidence. A later capture MUST produce a new source identity.
- **Failures:**

  | Code | Condition |
  |---|---|
  | `permission-denied` | Source release authority is absent. |
  | `unsupported` | The required terminal or isolation postconditions cannot be established. |
  | `invalid-argument` | Reference is invalid or foreign. |

- **Events emitted:** Established source stop and dependent publication termination are observable under Section 8.3.
- **Ownership:** Stop MUST NOT release a source without its owner's authority. Additional inseparable effects require Section 7.1 acceptance.
- **Cancellation and late completion:** Section 8.5 applies. Unconfirmed release MUST remain observable under Section 8.6.

**setSending(publication, enabled)**

- **Purpose:** Provide an optional compound sending convenience.
- **Inputs:** Publication, Boolean enabled intent and any applicable effect acceptance.
- **Preconditions:** The target is active and authorized. The requested compound behavior and isolation are supported.
- **Postconditions on success:** False sets a publication-scoped mute intent and records that setSending set it. True clears only the intent set by setSending(false); it MUST NOT clear an intent set by setMuted or setDisabled, or any other restriction. The effective publication mute is the union of the independent setMuted and setSending mute intents. Repeating either value is idempotent with respect to that intent. The implementation MUST preserve the distinction so a later true cannot undo an independent mute, including one set while setSending suppression is active. The result MUST identify established effects. Suppression MUST NOT be conflated with source release. Additional effects MUST be disclosed and accepted under Section 7.1.
- **Failures:**

  | Code | Condition |
  |---|---|
  | `invalid-state` | Publication is ended or otherwise ineligible. |
  | `permission-denied` | Required control or consent is denied. |
  | `unsupported` | The requested compound behavior or isolation cannot be established. |

  Partial and unknown effects MUST be reported under Section 9.
- **Events emitted:** Established changes to affected resources are observable under Section 8.3.
- **Ownership:** Existing ownership is unchanged. Acquired resources and accepted releases follow Sections 7.1 and 8.6.
- **Cancellation and late completion:** Section 8.5 applies. Late-acquired resources MUST be released or unresolved release reported.

An implementation MAY offer other compound conveniences. Before invocation it MUST disclose their constituent effects, ordering, partial outcomes and ownership. Each constituent contract MUST be preserved.

*Example:* Disabling a source suppresses both of its publications. Muting one publication while disabled and then enabling the source leaves that publication muted; the other can produce content if no other restriction applies.

*Rationale:* Publication mute and source disable represent independent intents. A single suppression flag cannot preserve a publication's mute when its source is re-enabled.

## 8. Session and Media Behavior

### 8.1 State transitions

These states describe locally established facts, not provider-wide state. A connecting session is observable only when the binding exposes a session handle before admission; join success returns a connected session. The tables enumerate application-visible transitions. An omitted transition is invalid. A state-changing event is delivered under Section 8.3 to observers whose coverage includes that change. An operation on a terminal resource MUST fail with `invalid-state`, except for idempotent release and inspection of unresolved cleanup.

| Client state | Trigger | Next state | Observable event | Notes |
|---|---|---|---|---|
| — | Client creation succeeds | `ready` | Ready client returned | Failed creation returns no usable client. |
| `ready` | Disposal takes effect | `disposed` | Disposal/terminal state | Child sessions and owned resources are handled under Section 8.6. |
| `disposed` | Repeated disposal | `disposed` | Operation completion; no new transition | Disposal is safe to repeat. |

| Session state | Trigger | Next state | Observable event | Notes |
|---|---|---|---|---|
| — | Join starts | `connecting` | Connecting state if exposed | A held admission remains pending. |
| `connecting` | Local endpoint admitted | `connected` | Connected state; join success | Admission does not establish media readiness. |
| `connecting` | Denial, cancellation, failure or local termination | `disconnected` | Terminal state and reason; join failure | Denial does not establish local participation. |
| `connected` | Connectivity interruption with recovery in progress | `reconnecting` | Reconnecting state | This does not establish departure. |
| `reconnecting` | Participation re-established | `connected` | Connected state | Recovery does not prove uninterrupted media or identity continuity. |
| `connected` or `reconnecting` | Local leave, removal, session end, failure, or termination of unestablished cause | `disconnected` | Terminal state and reason | Local leave does not request termination for everyone. |
| `disconnected` | Repeated leave | `disconnected` | Operation completion; no new transition | The terminal participation cannot resume. |

The disconnected reason MUST be `left`, `removed`, `ended`, `failed`, or `unknown`.

`left` denotes established local leave, `removed` established removal, `ended` established remote conference end, and `failed` an established failure including local connectivity failure. When simultaneous established causes cannot be ordered, any applicable established reason MAY be reported. Repeated observations without a state change are not transitions. Capture MUST report its established initial disable intent. The reason MUST reflect the established cause. `unknown` MUST be used when the cause is not established. New admission MUST require application authorization and revalidation.

| Participant presence | Trigger | Next state | Observable event | Notes |
|---|---|---|---|---|
| Not yet observed | Presence established | `present` | Presence included in snapshot or reported as added | A newly observed presence has a distinct identifier. |
| `present` | Observed departure or authoritative complete replacement excludes it | `departed` | Departure | Omission from an incomplete roster is not departure. |
| `present` | Recovery with continuity unestablished | Continuity unknown | Uncertainty exposed | Unknown continuity alone does not establish departure. |
| `departed` | A later presence is established | `departed` for the old presence; `present` for the new presence | New presence | The old presence never returns to active. |

| Source state | Trigger | Next state | Observable event | Notes |
|---|---|---|---|---|
| — | Capture succeeds | Live with its established disable intent | Source returned and available in covered state | Capture alone does not publish or send. |
| Live, enabled | Disable succeeds | Live, disabled | Disable state change | All dependent publications suppress source-derived content. |
| Live, disabled | Enable succeeds | Live, enabled | Disable state change | Production must be eligible, including any necessary reacquisition. |
| Live, disabled | Enable fails | Actual state reported, or unknown | Operation failure | The implementation MUST NOT claim restored production without evidence. |
| Live | Stop or owner teardown takes effect | `stopped` | Terminal source state and dependent publication endings | Section 8.6 governs release observation. |
| `stopped` | Repeated stop | `stopped` | Operation completion; no new transition | A stopped source cannot be enabled. |

| Publication state | Trigger | Next state | Observable event | Notes |
|---|---|---|---|---|
| — | Publish succeeds | Active with its established mute intent | Publication returned and available in covered state | Existing source disable intent is unchanged. |
| Active | setMuted(true) succeeds | Active; setMuted intent true | Intent and any effective mute change | The effective mute is the union of the setMuted and setSending mute intents. |
| Active | setMuted(false) succeeds | Active; setMuted intent false | Intent and any effective mute change | The setSending mute intent, source disable and other restrictions remain. |
| Active | setSending(false) succeeds | Active; setSending mute intent true | Intent and any effective mute change | The setMuted intent, source disable and other restrictions remain. |
| Active | setSending(true) succeeds | Active; setSending mute intent false | Intent and any effective mute change | The setMuted intent, source disable and other restrictions remain. |
| Active | Unpublish, source stop or owner teardown takes effect | `ended` | Publication ending | Native or remote deletion is not implied. |
| `ended` | Repeated unpublish | `ended` | Operation completion; no new transition | An ended publication never becomes active again. |

A source and each publication MUST retain their independent suppression intents while live. A failed state-changing operation MUST NOT be represented as a successful transition. Partial or unknown effects MUST be reported under Section 9. Any exposed readiness MUST identify its media scope and observed boundary. Join, selection, or render attachment alone MUST NOT establish active media.

### 8.2 Roster and identity

A roster MUST state its scope and completeness. Unknown completeness MUST be distinguished from complete and MUST NOT be treated as complete. Initial observed participants and publications MUST be available without reliance on future join events. Omission from an incomplete roster MUST NOT establish departure. An exposed count MUST agree with its stated scope and known members.

A participant identifier MUST remain stable for one presence. It MUST be unique and never reused within its session. Application identity, provider identity and endpoint identity MUST remain distinct. Application identity MUST be reported as authenticated only when established by the relevant authority. Otherwise its authentication status MUST be unknown. Display metadata MUST NOT authenticate identity. A delayed observation for a departed presence MUST NOT change a later presence. Across recovery, an identifier MUST be preserved only when continuity is established. When continuity is unknown, the implementation MUST expose that uncertainty. A newly observed presence MUST receive a distinct identifier. Unknown continuity alone MUST NOT be presented as departure. Identity MUST NOT be invented or silently randomized to resolve a collision.

Where an identity collision can affect admission or an existing presence, discovery MUST report the established collision policy or unknown. The implementation MUST NOT automatically retry a same-identity collision when that policy is unknown. A retry MUST NOT be represented as authorization to replace another presence. When continuity with a prior presence is unknown, retain the prior presence without asserting departure and expose the new observation with a distinct ID and an unknown-continuity association to the prior ID. If continuity is later established, expose the association without reusing either ID; otherwise do not merge them.

*Example:* A partial roster omits an earlier participant after reconnection. The omission does not establish departure. A newly observed endpoint whose continuity cannot be established receives a distinct presence identifier.

### 8.3 Snapshots and streams

Registration MUST atomically establish a detached snapshot at one coherent local observation point and a subsequent stream covering the observer's declared scope. A detached snapshot MUST NOT change when later state changes. For each established change in coverage, the initial snapshot or the subsequent stream MUST reflect it. The handoff MUST NOT cause the same change to be applied twice. The stream MUST deliver subsequently established covered changes in local commit order. Subscribers covering the same changes MUST see those changes in the same local commit order, including during reentrant callbacks. During notification, exposed public state MUST include the reported change. Related facts in one indivisible local observation MUST appear together. Separate provider facts MUST NOT be claimed to be atomic solely because they arrive in one callback. A later snapshot MUST reflect changes delivered before its observation point.

A local commit is the point at which a change becomes public state. An indivisible local observation is one public state transition that establishes related facts together; those facts appear together in its update.

A delivery MAY group changes only if each covered change and its order remain discernible to the application. Grouping MUST NOT erase an intermediate transition required by Section 8.1. Local delivery loss means a locally established covered change was not delivered, or a gap in that local delivery sequence was detected. The implementation MUST indicate loss before presenting a later change as a complete continuation. It MUST deliver a fresh snapshot or mark subsequent events non-current until the observer requests one. Provider-side history gaps and limitations in detecting them MUST be disclosed as coverage limitations. The fresh snapshot MUST establish a new gap-free handoff to subsequent changes under the rules above. The implementation MUST NOT claim to recover provider history it never observed.

*Note:* A binding may also offer a separate latest-value view; that view is not the ordered stream described here.

Unsubscribe MUST be idempotent. It MUST affect only the selected observer. No further delivery to that observer may occur after unsubscribe completes. Observed, last-known and unknown facts MUST remain distinguishable. Unknown MUST NOT be represented as false, zero, an empty complete collection, or established absence.

*Example:* If a participant joins while registration is establishing its snapshot, that presence appears either in the snapshot or in the following stream. If stream loss is detected later, a fresh snapshot restores current state; it does not reconstruct unobserved provider events.

### 8.4 Concurrency

Completed local state transitions affecting the same resource or coupled resources MUST be consistent with a serial order respecting real-time order of non-overlapping operations. A rejected conflict MUST have no conflicting effects. Cancellation or termination may interrupt an operation after effects; those effects MUST be reported under Section 9, and no later local transition may revive a terminated resource. Termination MUST remain possible while operations are pending and MUST NOT terminate a separately owned session. A pending prerequisite MUST settle or identify an active resolution mechanism and caller-controlled cancellation or deadline.

### 8.5 Cancellation and deadlines

The binding MUST declare a finite maximum time from receipt of a valid operation cancellation request to reporting whether cancellation won and, when it won, settlement of the cancelled operation. A cancellation request received while the result is not final MUST participate in the local cancellation/completion race without waiting for provider completion or native cancellation support. If completion wins that race, its original result remains final and cancellation reports that it did not win. It MUST declare a finite maximum time from expiry of a finite deadline to settlement. The binding MUST also declare finite maximum intervals from local termination taking effect to settlement of each fenced operation, and from invocation of leave or dispose to its settlement. These bounds apply without a caller deadline. The binding MUST document how clock suspension affects their measurement. The binding MAY use a caller-selected deadline with a declared finite settlement bound. A deadline MUST NOT expire before its stated duration. An operation MUST settle exactly once.

The `cancelled` code also covers an explicitly identified user dismissal, with reason `selection-dismissed`, distinct from an accepted operation cancellation that won.

| Race or condition | Result | Required subsequent behavior |
|---|---|---|
| Request is already cancelled before invocation | `cancelled` | No setup effects occur. |
| Cancellation is accepted before the operation result is final | `cancelled` within the bound | Do not wait for provider completion; prevent later success or public resurrection; initiate release of resources acquired by the operation. |
| Success or failure is final before cancellation is accepted | Original result remains final | Cancellation reports that it did not cancel the completed operation. |
| A finite deadline expires before the result is final | `timeout` within the bound | Apply the same fencing and cleanup rules as winning cancellation. |
| A resource is acquired after cancellation, deadline expiry, or termination fenced the operation | The settled result does not change | Initiate its release; expose unconfirmed release under Section 8.6. |
| Session termination occurs with an operation pending | Pending operation settles within the bound | It cannot subsequently report success or resurrect terminated state. |

Unknown remote effects and unresolved release MUST remain visible under Sections 8.6 and 9. A cancellation or timeout MUST NOT be reported as proof that remote effects did not occur.

*Example:* A capture request is cancelled before it settles, but the provider later returns a camera source. The operation remains cancelled and the implementation initiates release of that source.

### 8.6 Ownership and cleanup

A session owns its captured sources, publications and subscriptions. Teardown MUST stop its captured sources and end its publications and subscriptions, subject to the unresolved-release rule in Section 7.1. Teardown MUST NOT stop an application-owned source. Teardown MUST NOT release another owner's resources. Sharing a source does not transfer ownership or release authority.

Release MUST be idempotent. Release MUST remain available after support or authority changes to the extent the implementation retains release authority. Local release success MUST mean that the local handle is unusable and release has been initiated. Local release success MUST NOT claim confirmed remote or native release without observation. Teardown MUST NOT wait indefinitely for remote completion. Teardown MUST fence pending work as specified in Section 8.5.

A pending, failed or unknown release at closure MUST identify the affected resource or explicitly bounded scope. Its outcome, including later known changes and continuing uncertainty, MUST remain accessible after disposal for a documented retention period sufficient for the application to inspect it. A binding MAY offer an independently releasable observation handle; releasing that handle does not release the resource. Cleanup failure MUST be distinguishable from the initiating failure. When an operation creates a resource that is known to survive and that the application can control, the implementation MUST either provide a usable reference or release it. If release cannot be confirmed, it MUST report the release as unresolved. These obligations apply to resources acquired after settlement as well as before it.
*Rationale:* Local closure can precede provider release; retaining an unresolved outcome lets the application distinguish an unusable local handle from a confirmed release.

### 8.7 Receive and rendering

Core receive is provider-managed. Automatic receive MUST NOT be advertised as manual selection. A receive source MUST identify whether it represents one publication or an aggregate. Unknown contributors MUST remain unknown. An aggregate MUST NOT imply individually selectable publications.

Where a binding offers rendering, attachment MUST identify its source and target. Attachment acceptance MUST NOT be reported as playback. Detachment MUST NOT change capture, sending or recording. Binding-local playback mute MUST NOT change capture, sending or recording. Additional effects require disclosure and acceptance under Section 7.1. Teardown MUST release session-owned render attachments. Teardown MUST NOT stop an application-owned source.

### 8.8 Configuration

Each option MUST specify its operation, resource scope, activation boundary, mutability and known default or unknown. Initialization, provisioning, admission and runtime are distinct supply boundaries where supported. Omission MUST remain distinguishable from an explicit value. An accepted required option MUST be enforced throughout its stated scope or rejected before dependent effects. A preference MAY be unmet. A preference MUST be reported as selected, not selected, or unknown. A non-selected preference MUST include a reason for non-selection. An exposed effective value MUST be evidenced. Unknown MUST NOT be reported as a known value. Unknown option names, invalid values and wrong boundaries MUST reject before dependent effects. A recognized unsupported preference MAY be reported not selected with a reason when all required conditions hold; an unsupported required option MUST reject before dependent effects. Runtime changes MUST NOT silently trigger rejoin, recreation or renegotiation. Later mutation of input objects MUST NOT change accepted configuration.

Provider options MUST use a provider-qualified namespace and documented versioned schema.

This specification does not define a general catalog of portable option names. Bindings MUST document encoding and the type, scope, supply boundary and default of each portable option they expose; interoperable portable names require a specification defining them. A request MUST identify the schema version. Its meaning MUST NOT change during that request. Undeclared fields, types and determinable combinations MUST be validated before effects. An unknown namespace, field or version MUST be rejected. Conflicts with portable options MUST be rejected unless documented composition preserves both semantics. Schema validity MUST NOT be treated as support, authority or effectiveness.

## 9. Error Handling

Every failed portable operation MUST return a safe error with one core code. The table describes established conditions; retry safety also depends on the outcome and any effects, not on the code alone.

| Code | Definition and use | When not to use | Retry safety |
|---|---|---|---|
| `invalid-argument` | Input is malformed, inconsistent, misplaced or invalid. | Do not use for a valid request whose semantics are unsupported. | Correct the input first; effects must be assessed. |
| `invalid-state` | The target or operation is in an ineligible state. | Do not use merely because permission is denied or support is absent. | Retry only if the relevant state can change and effects permit it. |
| `unsupported` | The requested operation, combination or required condition cannot be provided. | Do not assert established lack of support solely from an unmapped failure. | Retry only after the relevant support or request changes. |
| `permission-denied` | Authority or required consent is denied. | Do not use for failure to authenticate a credential. | Retry only after authority or consent changes. |
| `authentication-failed` | Credential or identity authentication failed. | Do not use solely because credential acquisition failed. | Retry only with an appropriate credential or changed authentication conditions. |
| `cancelled` | An accepted operation cancellation won, or an explicitly identified user dismissal occurred with reason `selection-dismissed`. | Do not use if completion was already final or a deadline instead expired. | Check effects and cleanup before retrying. |
| `timeout` | An applicable deadline expired before settlement. | Do not use for a known network failure without deadline expiry. | Remote effects may be unknown; check before retrying. |
| `unavailable` | Network or service unavailability is established. | Do not use for a failure whose cause is unestablished. | Availability may change; check effects before retrying. |
| `internal` | An implementation failure is established. | Do not use as a catch-all for an unmapped provider failure. | No safety follows from the code; check effects. |
| `unknown` | No more specific portable condition is established. | Do not use in place of an established more specific code. | No safety follows from the code; check effects. |

The error MUST include a safe message. It MUST include an outcome. It MUST identify established reasons. A sanitized provider code MAY be included separately from the portable code. A provider code or message MUST be sanitized before exposure. Default errors, events, snapshots, logs and serialization MUST exclude credentials, secret-bearing URLs and headers, key material, raw provider payloads and stacks, subject only to the authorized diagnostic exception in Section 12.1. Diagnostic formatting MUST handle cycles and non-error values without exposing their unrestricted contents. Unrestricted diagnostics, if offered, MUST require an explicit request and remain within its authorized boundary.

The closed core reason vocabulary is `credential-acquisition-failed`, `credential-expired`, `credential-revoked`, `authentication-failed`, `authorization-denied`, `operation-conflict`, `resource-not-found`, `resource-conflict`, `quota-or-rate-limit`, `network-failure`, `service-unavailable`, `media-failure`, `protocol-failure`, `cleanup-failure`, `selection-dismissed`, and `unknown`. The credential reasons distinguish failure to obtain a credential, established expiry, and established revocation. `authentication-failed` denotes failed authentication; `authorization-denied` denotes denied authority or consent. `operation-conflict` denotes a conflicting operation; `resource-conflict` denotes a conflicting resource condition; `resource-not-found` denotes an established absent resource. `quota-or-rate-limit` denotes an established quota or rate limit. `network-failure`, `service-unavailable`, `media-failure`, and `protocol-failure` identify their respective established failure domains. `cleanup-failure` denotes an established release failure. `selection-dismissed` denotes an explicit user dismissal without selection. `unknown` denotes an unestablished reason. An error MUST identify all applicable reasons known at settlement. Include `unknown` when no specific reason is established; known reasons do not imply that all causes are known. Credential acquisition failure MUST remain distinct from rejection of a supplied credential. Renewal failure alone MUST NOT assert disconnection. The implementation need not discover additional reasons solely to rank them. No total precedence among simultaneous causes is required.

If multiple conditions are established, any applicable core code MAY be returned, with all known applicable reasons. Applications MUST NOT infer the absence of another cause from that code. Profile reasons and codes MUST be namespaced. An unrecognized profile code MUST retain an interoperable core code.

An error rejecting a supplied option MUST identify its complete path. It MUST identify the operation to which the option was supplied. It MUST distinguish invalid value or shape from unsupported semantics. A recognized option at the wrong operation boundary MUST be rejected as invalid.

| Rejected required capability or placement condition | Assessment | Additional requirement |
|---|---|---|
| Known not to be provided | `unsupported` | Identify the rejected condition. |
| Not established | `unknown` | Identify the rejected condition. |
| Evaluation can resolve through an identified mechanism | `pending` | Identify the resolution mechanism and evaluation boundary. |

Each rejected required condition MUST be identified with its assessment. Known denial, invalid input, or an established unsatisfied requirement MUST NOT be reported as pending. Placement rejection MUST identify the requested scope and boundary. No dependent effect may occur while a required condition is unsatisfied or unknown.

A request with a pending required condition MAY remain pending only while its resolution mechanism is active and cancellation and deadline rules apply. If it cannot or will not wait, it MUST fail `unsupported` with the pending assessment, resolution mechanism and evaluation boundary, without dependent effects.

| Outcome | Meaning |
|---|---|
| `none` | No attributable effect occurred. |
| `partial` | At least one attributable effect occurred and the full postcondition did not. |
| `unknown` | Neither `none` nor `partial` is established. |

A result MUST use `none` only when absence of attributable effects is established.

An attributable effect is a public or external change caused by the operation, including a known remote change; a later reversal does not erase its occurrence. Where details are pending, the result MUST identify how to obtain later updates and the documented retention period under Section 8.6. The result or an accessible subsequent update MUST identify known completed effects and known surviving resources. It MUST identify effects whose occurrence is unknown. It MUST distinguish independently requested steps that were not started. It MUST NOT claim rollback without evidence. Known surviving application-controllable resources MUST be usable or released as specified in Section 8.6. An operation error MUST be distinguishable from an unsolicited error observation. When they are known to describe the same failure, they MUST be correlatable. The original failure and cleanup failures MUST remain distinguishable.

Retry advice, if exposed, MUST identify the operation and its evidence. Retry advice MUST NOT confer authority. Retry advice MUST NOT establish absence of effects. An implementation MUST NOT automatically retry an operation when doing so could duplicate an unknown remote effect.

*Example:* A publish request times out after the provider may have created a publication. Its outcome is `unknown` unless the effects are established. Automatic retry is prohibited if it could create a second publication.

*Example:* A valid option requesting an unsupported mode is `unsupported`; a malformed value is `invalid-argument`. Both identify the option's complete path.

## 10. Extensibility

A profile MUST define a unique name and version, discovery owner and scope, compatibility and dependencies, operations, states, success and failure, ownership, authorization, partial execution, security and cleanup. A profile claim MUST satisfy its defining operations and requirements. Discovery MUST have no protected effects or consent prompts. Discovery MUST distinguish unsupported from unknown. Discovery MUST identify the supported version or range and how changes become observable. An incompatible required version MUST fail before dependent effects. Preferences MUST NOT weaken requirements. A weaker alternative MUST NOT be selected without explicit acceptance.

Provider extensions MUST use reverse-DNS-style namespaces controlled by their owner and documented versions. An owner MUST NOT use a namespace controlled by another owner. A request MUST identify the applicable version. The interpretation of that version MUST NOT change during the request. An extension MUST define its scope, operations, ownership and outcomes. The implementation MUST validate the whole request, including types, unknown fields and combinations, before determinable effects. Unknown namespaces, fields, versions and wrong operation boundaries MUST be rejected. Conflicts with portable options MUST be rejected unless documented composition preserves both requested semantics. Schema validity MUST NOT imply support, authority or effectiveness. An extension MUST NOT alter core operation semantics or confer authority. Core security, errors, lifecycle and cleanup apply. Discovery and acquisition MUST reject malformed or incompatible required versions before acquisition.

## 11. Optional Profiles

Each following profile defines version **1** and is discovered on its stated owner through Section 7.3. Each profile inherits Sections 7.1, 8.4–8.6 and 9, including cancellation, coupled-effect disclosure, partial outcomes, ownership and cleanup. Each invocation MUST recheck current support and authority. Invalid targets and states MUST fail before effects. Unsupported, unauthorized, unavailable and other failures use the established Section 9 code and reason. A profile claim includes every operation mandatory in Section 7.0. Explicitly optional features MUST be separately advertised. Required conditions MUST be assessed together before dependent effects. A capability does not grant authority.

### 11.1 Calling

Calling version 1 is discovered on the client. A **Call** is a client-owned local calling interaction with an opaque scoped identity, direction `incoming` or `outgoing`, state, peer information and independent local and remote hold observations. An unknown peer or hold observation MUST remain unknown. A Call is distinct from a conference session and MUST NOT fabricate conference participants, publications or membership. Dialing and answering MUST NOT require conference provisioning or join. A call's signaling address or caller ID MUST NOT be represented as authenticated application identity without supporting evidence. Call destinations, caller identities and digits are sensitive. Access to them MUST be authorized. Signaling and diagnostics MUST protect their confidentiality within the applicable trust boundary. The client owns calls it creates or accepts. A binding providing call media MUST provide an authorized call-scoped acquisition path or accept an existing source with its owner’s grant; it MUST NOT represent a Call as a conference Session to do so. Termination releases call-owned resources and MUST NOT stop application-owned sources without their owner's authority.

**captureCall(call, kind, options)** acquires a call-owned source; **publishCall(call, source, options)** publishes an eligible source to the call. They follow the capture and publish requirements in Section 7.4 with the Call as their scope and owner in place of the Session. Use of a source owned by another party requires its owner’s grant. Neither operation requires a Session. A publication returned by publishCall is owned by the Call; a publication returned by publish is owned by the Session. The source retains its existing owner.

Calling, incoming delivery, and each optional control MUST be discovered for their actual scope and prerequisites. Authority to dial or answer MUST NOT imply authority for hold, transfer, digits or registration. Applicable application request, owner authorization and platform or user consent MUST precede capture or sending. Required policy, placement and protection MUST be enforceable before affected signaling or media. Required protection MUST be effective before protected early media or other protected content flows.

Discovery MUST identify remote-initiated update flows and enforceable decisions. The application selects a remote-update policy; the default MUST reject the entire update when it adds a media kind. A separately selected exclusion policy accepts only an answer excluding the new kinds; it is not whole-update rejection. If the selected policy cannot be enforced, setup enabling that flow MUST fail before effects. The decision and established added and removed kinds, or unknown, MUST be observable. Consent and required protection apply before affected media effects.

A call observation supplies a current snapshot and subsequent changes with the handoff, ordering, loss and resynchronization guarantees of Section 8.3. Changes to state, direction, peer identity, media availability, hold, transfer progress and terminal reason MUST be observable when established. A signaling status, when exposed, MUST remain distinguishable from observed media availability and packet or playback readiness. No observed media or readiness MUST be inferred solely from ringing, provisional signaling, request acceptance or answer.

The following states describe locally established call facts. `ringing` means remote alerting for an outgoing call or an offered unanswered incoming call. `early-media` means media is established or offered before answer; its direction and availability MUST be qualified as known or unknown. `held` and `transferring` are overlays on an established call: independent hold and transfer observations remain available even when both apply. A binding MUST represent an unknown hold separately from false. Transfer activity is independently represented; hold changes during transfer update the hold observations without ending transfer. A single displayed state gives transferring priority while transfer is active; otherwise it is held if either hold is true, active only if both are known false, and unknown hold is separately exposed. A transfer request alone MUST NOT end the call. For an established answered Call without an established active transfer, when neither hold is known true and either hold is unknown, the single displayed state is unknown. This does not undo the established answer or imply termination; the independent hold observations retain the uncertainty.

| Current state | Established trigger | Next state or observation |
|---|---|---|
| none | outgoing dial accepted | `dialing` |
| none | incoming offer delivered | `ringing`, direction incoming |
| `dialing` | remote alerting | `ringing` |
| `dialing` or `ringing` | pre-answer early media observed | `early-media` |
| `early-media` | early media ends while unanswered | `ringing` if alerting or incoming offer remains, otherwise `dialing` |
| `dialing`, `ringing` or `early-media` | answer established | `transferring` if transfer is active; otherwise `held` if either hold is true, `active` if both are known false, or `unknown` with the answer retained |
| `active`, `held` or `unknown` (answered Call) | local or remote hold established or changed | `held` while either hold is true; otherwise `active` if both are known false, or `unknown` with the answer retained |
| `active`, `held` or `unknown` (answered Call) | transfer request accepted | `transferring`, retaining independent hold observations |
| `transferring` | transfer completes, fails or becomes unknown while call remains live | `held` if either hold is true; `active` if both are known false; otherwise `unknown` with the answer retained |
| any nonterminal state | local rejection or hangup, remote end, failure or other termination established | `ended` |
| `ended` | repeated hangup | `ended` |

When neither hold is known true and either is unknown, the implementation MUST expose that uncertainty rather than assert an unheld state. An unanswered incoming call in early media remains eligible for accept or reject until an answer attempt starts. A failure of one underlying signaling branch MUST NOT end a Call while its aggregate call remains live. A terminal Call MUST NOT resume. A new call or a replacement signaling interaction MUST have a distinct Call identity. A changed provider identifier alone MUST NOT establish replacement when continuity is evidenced.

The terminal observation MUST identify the established reason as `local-rejected`, `remote-rejected`, `local-hangup`, `remote-hangup`, `failed`, `transferred`, or `unknown`. `transferred` requires evidence of successful transfer. A provider's more specific sanitized reason MAY also be exposed. An unestablished cause MUST be `unknown`. Local closure MUST NOT claim confirmed remote release without evidence.

**dial(client, destination, options)** requires a ready client, a valid destination and current calling authority. Success returns an outgoing Call in `dialing` and means the request was accepted, not answered. An answer or termination, if established, MUST be observable; no bounded answer is promised. Dial MUST NOT implicitly capture or publish. Explicit bundled media requires Section 7.1 disclosure and applicable consent. A required policy that cannot be verified fails before affected effects.

**Incoming delivery** exposes an incoming Call before application acceptance or rejection. Delivery alone MUST NOT answer, capture or send. The observation MUST distinguish an unverified caller identity from an authenticated identity. An incoming Call may end remotely before application action. The implementation MUST NOT deliver a late invitation after its owning client is disposed.

**accept(call)** requires an unanswered incoming Call and answer authority. Success means acceptance of the answer request, not an established answer or media. An answer or termination, if established, MUST be observable; no bounded answer is promised. A concurrent conflicting accept MUST fail `invalid-state` with reason `operation-conflict` before effects. An accept after an established answer or call end MUST fail `invalid-state`. After a failed attempt, another attempt is allowed only while the call remains unanswered and the provider permits it. Required media, consent and protection checks apply before affected effects.

**reject(call)** requires an unanswered incoming Call and rejection authority. Success means the local rejection request was accepted and the local Call is ended. Reject MUST NOT create an answer attempt. Optional `busy` and `declined` reasons MUST be separately discoverable. An unsupported requested reason MUST fail before effects. A requested reason MUST NOT promise a particular remote presentation. An omitted reason uses the documented provider default.

**hangup(call)** requires authority to terminate the Call. Success ends the local Call and initiates release of call-owned resources. Repeated hangup MUST succeed as a no-op. Local closure MUST NOT be presented as confirmed remote release. Pending operations and unresolved release follow Sections 8.5, 8.6 and 9.

**setHold(call, held)** requires an established Call and authority for the requested hold or resume. Success reports the established local hold state; request acceptance alone MUST NOT be reported as established hold. While negotiation is in progress the operation remains pending, subject to the settlement bounds in Section 8.5 when cancellation is accepted, a finite deadline expires, or termination occurs. Otherwise failure or an unknown negotiated outcome MUST be reported without inventing the requested state. Local and remote hold MUST be observed independently. Local resume MUST NOT clear or imply the end of remote hold. Hold or renegotiation alone MUST NOT imply disconnection or media failure. Conflicting operations follow Section 8.4.

**transfer(call, destination)** is optional and separately advertised. A destination transfer and an attended transfer using another Call MUST be advertised separately when offered. An attended transfer requires a distinct established consultation Call owned by the same client. Unsupported combinations MUST fail before signaling. Success reports request acceptance and provides observation of progress and eventual completed, failed or unknown outcome. Acceptance MUST NOT imply completion or termination of either Call. Source Call termination alone MUST NOT establish transfer completion. If the source Call ends before the transfer outcome is final, the last established outcome MUST be reported within the bound in Section 8.5. If no outcome has been established, report `unknown`. Resulting Calls and their ownership MUST be exposed when known. A resulting outgoing Call MUST have a distinct identity and MUST NOT be represented as an incoming offer. A source Call remains live until independently ended.

If incoming transfer execution is supported, discovery MUST identify its effective policy and whether interception is enforceable before enabling it. The implementation MUST NOT execute an incoming transfer without an explicitly selected policy authorizing execution. An application-controlled request MUST remain pending until explicitly accepted or rejected. A timeout of an application-controlled request MUST reject it without execution. A policy that cannot be enforced MUST fail before the affected setup or execution. Required client, placement and protection conditions apply to any resulting Call before dependent effects.

**sendDigits(call, digits, methods)** is optional and separately advertised from incoming digit observation. Discovery MUST identify supported digits, methods, timing constraints and prerequisites for the target Call. The request MUST identify a primary method and any explicitly authorized ordered fallback methods.

The methods list MUST be nonempty: its first element is the primary method and remaining elements are ordered authorized fallbacks. The submitted prefix is the exact leading subsequence of the requested digits, in request order. Timing parameters and units MUST be documented by the binding. An omitted or empty fallback list authorizes only the primary method. The implementation MUST validate the requested sequence and policy before submission. It MUST use only authorized methods in the requested order. It MUST NOT retransmit an already submitted digit through fallback. If these restrictions cannot be enforced, the request MUST fail `unsupported` before submission.

The send result or an accessible subsequent update MUST identify the exact known contiguous submitted prefix or state that it is unknown. It MUST identify the methods used in first-use order or state that they are unknown. It MUST report local sequence completion as complete, partial, failed or unknown. Complete requires evidence that the whole sequence was submitted. Failure, cancellation or timeout MUST retain known or unknown partial submission as required by Section 9. Submission MUST NOT imply remote consumption. A generic message or locally audible tone MUST NOT be reported as telephone digit submission.

**Incoming digit observation**, when supported, identifies the Call and observed digits. Its readiness MUST NOT imply remote reachability or sending support. Absence of an observed digit MUST NOT establish that none was sent. Removing an observation MUST be idempotent and MUST prevent further delivery to that observer after completion.

**Receiving registration**, when supported, is separately advertised on the client. Discovery MUST identify account scope and how registration changes are observed. Starting reception MUST require an explicit request and account-scoped credentials. Acquiring the profile or observing incoming calls MUST NOT itself register an account. Start succeeds only when local invitation delivery is enabled and registration or listener readiness is evidenced. The observation MUST distinguish inactive, starting, receiving, stopping, failed and unknown. It MUST distinguish known validity, expiry and refresh activity from unknown, and use not-applicable for registration validity on a registrationless listener. Refresh MUST NOT invalidate still-evidenced readiness. Expiry without evidenced renewal MUST NOT be represented as valid registration. Readiness loss and registration failure MUST be observable. Automatic re-registration requires an explicitly selected policy. Receiving readiness MUST NOT imply external routing, NAT traversal, remote reachability or future call success.

**startReceiving(client, account, options)** and **stopReceiving(client, account)** represent the start and stop operations in a binding; equivalent names and parameter shapes are permitted. Stopping reception MUST identify the account or listener being stopped. Stopping reception MUST fence further incoming delivery for that scope and initiate release of its owned registration resources. Repeated stop while inactive MUST be a no-op. Stopping reception MUST NOT terminate existing Calls. If preservation of existing Calls cannot be established, stop MUST fail `unsupported` before effects. Client disposal MUST stop owned reception and fence late invitations. Registration release follows Section 8.6.

*Example:* A callee hears an announcement during early media and then receives a rejection. The Call was never active merely because audio was available. A transfer request accepted before the Call ends can still have an unknown outcome.

*Rationale:* Request acceptance, negotiated state and remote completion occur at different observable boundaries; treating them as the same would misreport answered calls, successful holds or completed transfers.

### 11.2 Manual Subscription and Receive Control

Version 1 is discovered on the session for the specific remote publication and media kind. Automatic provider-managed receive MUST NOT be advertised as manual selection. Selection requires current provider authorization and does not grant it.

**subscribe(session, publication)** requires a live session, a currently identifiable remote publication in that session, support for its kind and authority to receive it. Success returns a session-owned Subscription in `selected` state for that publication. Selection means accepted manual selection for receipt, not packets, decoding, playback or indefinite retention. A duplicate selection for the same publication and kind while pending or selected MUST fail `invalid-state` with reason `operation-conflict` before effects. Unavailable or unauthorized selection MUST fail without selecting. Partial or unknown effects follow Section 9.

**unsubscribeMedia(subscription)** removes that Subscription's manual selection. It MUST be idempotent. It MUST NOT remove the remote publication or stop its source. Any other policy or binding that retains reception MUST be disclosed as an additional effect or retained condition. It MUST NOT be represented as proof that all reception has stopped.

| State | Trigger | Next |
|---|---|---|
| `selected` | unsubscribe, remote publication end or session teardown | `ended` |
| `ended` | repeated unsubscribe | `ended` |

An ended Subscription MUST NOT resume. A later selection creates a distinct Subscription. State changes MUST be observable under Section 8.3. Session teardown ends its Subscriptions under Section 8.6. Access to media remains subject to provider authorization and applicable privacy protections.

### 11.3 Source Replacement and Sharing

Version 1 is discovered on the session and source, including the proposed recipient or publication scope. Each operation requires current support, applicable consent and authority over every affected resource. A grant does not confer authority beyond its stated scope.

**shareSource(source, recipient, grantScope)** requires a live source and its owner's explicit authorization. Success returns an active Grant identifying its owner, recipient, source and permitted scope. The source owner owns the Grant. The Grant permits use only within that scope. It MUST NOT transfer source ownership or stop authority. A recipient MUST NOT extend or transfer the Grant without the owner's explicit authorization.

**revokeSourceGrant(grant)** requires the Grant owner's authority. Success prevents further use under that Grant. Any coupled effects MUST be disclosed and accepted under Section 7.1 before execution. Revocation MUST NOT be reported as undoing delivered media. The source owner's teardown or stop and recipient teardown retain the ownership and cleanup rules of Sections 7.4 and 8.6.

| Grant state | Trigger | Next |
|---|---|---|
| `active` | successful revocation or source termination | `revoked` |
| `revoked` | repeated revocation | `revoked` |

A revoked Grant MUST NOT become active again. A new authorization creates a distinct Grant. Grant changes MUST be observable. A recipient's teardown MUST NOT stop a source owned by another party.

**replaceSource(publication, source, requireIdentity)** requires an active publication, a live eligible replacement source and authority over both. Use of a source owned by another party requires a grant covering the use. The operation MUST preserve publication mute intent and source disable intent. It MUST respect both source owners. A required publication identity continuity MUST fail before effects if continuity is unverified. Success MUST report publication identity as preserved, changed or unknown. If replacement uses unpublish and publish, old removal and the new publication MUST be observable and the operation MUST NOT be described as atomic. An operation MUST NOT report success before the promised replacement state is observable. Partial or unknown results MUST identify known surviving resources and unknown effects under Section 9. The original publication remains active or ends according to the established result. Replacement MUST NOT stop either source unless separately authorized and disclosed under Section 7.1.

*Example:* Replacing a muted publication with an enabled camera leaves the publication muted. Re-enabling a disabled replacement source does not clear the publication's mute intent.

### 11.4 Screen Sharing

The following failure table applies to each operation in Sections 11.4–11.11, in addition to operation-specific mappings and Section 9. A condition is reported only when established. Validation and known authorization failures precede effects.

| Condition | Core code |
|---|---|
| Malformed, inconsistent or misplaced input | `invalid-argument` |
| Ineligible target or state | `invalid-state` |
| Unsupported operation, combination or required condition | `unsupported` |
| Denied authority or required consent | `permission-denied` |
| Failed credential or identity authentication | `authentication-failed` |
| Accepted cancellation wins | `cancelled` |
| Applicable deadline expires | `timeout` |
| Network or service unavailable | `unavailable` |
| Established implementation failure | `internal` |
| No more specific condition established | `unknown` |

Version 1 is discovered on the session. Discovery identifies supported display surfaces, screen audio, simultaneous sending and receiving, platform prerequisites, and coupled effects, or identifies each as unknown. Discovery does not prompt for consent. Generic video capture support does not imply screen capture support.

**captureScreen(session, options)**

- **Purpose:** Acquire the selected display content as a source.
- **Inputs:** A session and options identifying requested surface and any required audio or isolation.
- **Preconditions:** The application explicitly requests capture. The session is eligible and authorized. Applicable platform display consent is obtained before protected use. A pending consent interaction is not consent.
- **Postconditions:** Success returns a live session-owned screen source for the established selection. Capture alone does not publish. Publication uses Section 7.4. The result identifies the actual selection scope or states it is unknown.
- **Failures:** The shared failure table in Section 11.4 applies. The table below applies in addition to Section 9.
- **Events:** Source and publication observations reflect established user or platform capture termination and subsequent selection-scope changes.
- **Ownership:** The session owns the returned source. Stopping it follows Sections 7.4 and 8.6.
- **Cancellation:** Section 8.5 applies, including release or unresolved-release reporting for late capture.

| Condition | Result |
|---|---|
| Selection dismissed without a selection | `cancelled` with reason `selection-dismissed`; no capture success is reported |
| Consent denied | `permission-denied`; no protected use occurs |
| Required surface, audio or isolation unsupported or unverifiable | `unsupported` before dependent effects |
| Failure after acquisition | Section 9 outcome and Section 8.6 cleanup apply |

The source states and transitions are those of Section 8.1. A revoked grant ends or disables capture as established. Silent reacquisition MUST NOT bypass consent. Stopping a share MUST NOT silently stop an unrelated camera or microphone. Screen contents can include unrelated private information. Selection scope and changes MUST be represented truthfully.

### 11.5 Data Messaging

Version 1 is discovered on the session. Discovery identifies recipient scopes, eligibility and membership boundary, maximum payload size in octets, supported reliability and ordering classes, their delivery context and the conditions ending retransmission eligibility. Unknown constraints are identified as unknown. Messages are ephemeral byte sequences.

**sendMessage(session, recipients, payload, options)**

- **Purpose:** Submit one message to the specified eligible recipients.
- **Inputs:** Session, explicit recipient scope, immutable payload bytes and requested reliability and ordering.
- **Preconditions:** The sender has authority. The session and recipient references are eligible. The complete requested combination is supported and its required properties established.
- **Postconditions:** Success identifies established local submission and its acceptance boundary. It does not establish remote receipt or consumption. It identifies established per-recipient outcomes or unknown outcomes.
- **Failures:** The shared failure table in Section 11.4 applies. The table below applies in addition to Section 9.
- **Events:** Established delivery or failure and incoming messages are observable within the declared coverage. A missing delivery event is not proof of failure.
- **Ownership:** The session scopes sending and observation. Sending does not transfer ownership of the application's payload.
- **Cancellation:** Section 8.5 applies. Known and unknown submissions remain reported even when cancellation wins.

| Condition | Result |
|---|---|
| Payload exceeds the applicable advertised limit | `invalid-argument` before sending |
| Requested recipient scope, reliability or ordering cannot be enforced | `unsupported` before sending |
| Recipient is unauthorized | `permission-denied` before effects when determinable |
| Some submissions occur or their occurrence is unknown | Section 9 identifies known and unknown effects and unstarted recipients |

Discovery MUST identify the applicable maximum message size in payload octets before transport encoding, or report it unknown. The implementation MUST reject an over-limit message before sending and MUST NOT truncate it. If the applicable limit is unknown, a request requiring a known limit follows Section 7.3. A direct recipient set MUST be nonempty and identify current endpoint presences. A broadcast scope MUST identify the eligible endpoints and membership boundary. An unsupported direct send MUST NOT become a broadcast.

| Class | Observable contract |
|---|---|
| Best-effort | Submission does not promise retransmission or delivery. |
| Reliable | Retransmission is provided within the disclosed delivery context until its disclosed eligibility ends; it does not promise delivery beyond that context, durable storage or exactly-once delivery. |
| Ordered | Delivery order is guaranteed only within the disclosed scope and context. |
| Unordered | No delivery order is promised. |

An implementation MUST NOT claim ordering outside its declared scope. It MUST NOT retain payloads as session history or replay sends after recovery. The implementation MUST NOT queue messages for offline recipients. Durable or delayed messaging requires a separate profile, which this specification does not define. It MUST NOT automatically retry an unknown submission where duplication is possible.

**observeMessages(session)**

- **Purpose:** Observe incoming messages within the stated coverage.
- **Inputs:** Session and any supported observation scope.
- **Preconditions:** The observer is authorized for that scope.
- **Postconditions:** The stream identifies each observed message's sender identity as established or unknown and its actual recipient scope. Unknown sender identity is not authenticated.
- **Failures:** The shared failure table in Section 11.4 applies. Invalid scope, unsupported observation and denied authority use Section 9. Loss or possible loss is observable.
- **Events:** Incoming messages and stream loss are delivered within the stated coverage. Section 8.3 governs applicable state observation and resynchronization.
- **Ownership:** The observer owns its observation handle. Unsubscribe follows Section 7.3.
- **Cancellation:** Section 8.5 applies to asynchronous setup. Unsubscribe prevents further delivery after completion.

| Message outcome | Evidence needed | Later outcome |
|---|---|---|
| `submitted` | Local submission established | `delivered`, `failed`, or unknown when established |
| `delivered` | Delivery evidence for the stated recipient and boundary | No inference of consumption |
| `failed` | Failure established for the stated recipient and boundary | No inference about other recipients |
| unknown | Neither delivery nor failure established | An established later observation may refine it |

Payload and metadata require access control and appropriate transport protection. Payloads MUST NOT appear in retained session snapshots, diagnostics or default logs. Encryption coverage is disclosed separately under Section 11.8.

*Example:* A provider acceptance receipt establishes submission at that provider boundary. It does not establish that a recipient received or read the message.

### 11.6 Recording

Version 1 is discovered on the session for session-scoped recording and on the trusted server for independent recording jobs. Discovery identifies available ownership models, recordable content, processing and storage scope, encryption compatibility, notification and consent mechanisms, and who can observe recording state. Unknown properties are identified as unknown. Every recording identifies its ownership model: a session-scoped recording is owned by its initiating session, whose leave initiates its stop; an independent recording job is owned by the trusted server under Section 11.9 and survives local participant departure. Departure does not imply deletion of either recording output.

**startRecording(session, options)**

- **Purpose:** Start recording the identified session content.
- **Inputs:** Session, content and processing/storage scope, and applicable options.
- **Preconditions:** The application explicitly requests recording. The initiating owner has recording authority. Required participant notice and consent are obtained before recording begins. Pending or denied consent is not permission.
- **Postconditions:** Success returns a session-scoped recording owned by the authorized initiating local owner only after start is established. The result identifies its scope. A request acceptance alone does not establish recording.
- **Failures:** The shared failure table in Section 11.4 applies. The table below applies in addition to Section 9.
- **Events:** Authorized observers receive established recording state changes and loss of knowledge. The profile identifies which participants or other principals can observe state and the coverage of that observation.
- **Ownership:** The initiating local owner owns the session-scoped recording. Its scope and stop authority are identified. Its leave ends the recording and initiates release under Section 8.6.
- **Cancellation:** Section 8.5 applies. A surviving job is identified or released under Sections 8.6 and 9.

| Condition | Result |
|---|---|
| Required notice or consent denied | `permission-denied` before recording |
| Required notice or consent remains pending | No recording until satisfied; pending has a resolution mechanism and boundary |
| Required encryption or placement incompatible or unverified | `unsupported` before the affected effect, with rejected conditions under Section 9 |
| Start partly succeeds or is unknown | Report known surviving jobs, unknown effects and unresolved release under Sections 8.6 and 9 |

The profile MUST specify how required notice and consent are obtained and which participants they cover. Recording MUST NOT silently weaken required encryption or placement.

**startIndependentRecording(server, conference, options)** is a Server Administration operation, separately discovered and authorized on the trusted server. It has the start, notice, consent, scope, protection, observation and failure obligations above. Success returns an established independent recording job owned by the trusted server. Participant leave MUST NOT end this job. Its stop authority, observation and cleanup remain available independently of any participant session.

**stopRecording(recording)**

- **Purpose:** Stop the identified recording.
- **Inputs:** Recording reference.
- **Preconditions:** The caller has stop authority. An already stopped recording is eligible for repeated stop.
- **Postconditions:** Success initiates local stop and closes the control handle to new work; a separate authorized observation remains available under Section 8.6. `stopped` requires evidence of job termination. Repeated stop is safe. Request acceptance does not establish terminal job or output completion.
- **Failures:** The shared failure table in Section 11.4 applies. Denied authority, unavailable service and other established conditions use Section 9. Partial or unknown stop identifies surviving jobs and unresolved release.
- **Events:** Established stop and later established cleanup outcomes are observable.
- **Ownership:** Stop affects only the identified recording and authorized scope.
- **Cancellation:** Section 8.5 applies. Cancellation does not imply that a submitted stop had no remote effect.

| State | Established observation | Next |
|---|---|---|
| `recording` | Stop established | `stopped` |
| `recording` | Current state not established | `unknown` |
| `unknown` | Recording or stop established | `recording` or `stopped`, respectively |
| `stopped` | Repeated stop | `stopped` |

Unknown is not proof of stop. Recording files and metadata require access control and retention appropriate to their sensitivity.

The recording service policy MUST identify authorized principals and access, retention and deletion boundaries for recording files and metadata. Local stop does not prove deletion of remote outputs.

### 11.7 Placement

Version 1 is discovered for the resource and activity. Discovery identifies the profile-defined set of covered resource classes and applicable processing, storage, signaling, media, deployment or data scopes. These are scope descriptions, not a mandatory core enumeration. Discovery identifies supported boundaries, transit and fallback coverage, or unknown.

**assessPlacement(scope, requirements)**

- **Purpose:** Assess the complete placement request without protected effects.
- **Inputs:** Resource scope and requirements and preferences, including inherited constraints.
- **Preconditions:** The request identifies each covered resource class and applicable activity. Allowed regions use namespace-qualified identifiers. The request states whether transit and fallback are included.
- **Postconditions:** The assessment covers the complete combination. It distinguishes supported, unsupported and unknown as in Section 7.3.

Capability assessment alone MUST NOT be reported as current enforcement. Activity classes include processing, storage, signaling, media, deployment and data; the request identifies the resource and activity boundaries to which each applies. Preferences report selected, not selected or unknown independently of observed location.
- **Failures:** The shared failure table in Section 11.4 applies. Invalid requests and rejected required conditions use Section 9. Rejection identifies each condition, its assessment, scope and boundary.
- **Events:** Known changes in capability or enforcement are observable as in Section 7.3.
- **Ownership:** Assessment acquires no affected resource and changes no ownership.
- **Cancellation:** Section 8.5 applies to asynchronous assessment.

Requirements apply conjunctively to each covered resource, including inherited requirements. Overlapping allowed sets intersect. An empty intersection rejects before dependent effects. Region identifiers from different namespaces are not equivalent without an established mapping. Preferences MUST NOT weaken requirements. Each placement entry identifies whether it is required or a preference. Only required entries participate in mandatory intersections. Each preference result MUST identify its corresponding request entry and report selected, not selected with a reason, or unknown, independently of required enforcement.

**applyPlacement(scope, requirements)**

- **Purpose:** Establish and maintain the requested placement enforcement for the stated scope.
- **Inputs:** Resource scope and placement requirements and preferences.
- **Preconditions:** The caller has authority. The complete required combination is established as enforceable before the first covered effect.
- **Postconditions:** Success identifies the enforcement state and exact boundary. Enforcement persists throughout use, recovery and fallback. A stored or transferred session policy preserves every applicable required constraint and scope.
- **Failures:** The shared failure table in Section 11.4 applies. The table below applies in addition to Section 9.
- **Events:** Changes in verification and affected boundaries are observable.
- **Ownership:** Affected resources retain their existing owners and cleanup obligations.
- **Cancellation:** Section 8.5 applies. Partial or unknown effects and surviving resources are reported under Section 9.

| Condition | Result |
|---|---|
| Requirement unsupported, unknown or unverifiable | Reject before dependent effects with Section 9 assessment and boundary |
| Verification lost or contradicted during use or recovery | Report affected boundary; prevent further implementation-controlled out-of-bound effects |
| Fallback cannot satisfy a requirement | Do not perform the affected out-of-bound fallback |
| Enforcement or effects become unknown after execution starts | Report unknown and known effects under Section 9; do not claim verified enforcement |

| State | Meaning | Transition |
|---|---|---|
| `verified` | Current enforcement is established for the stated boundary | Becomes `unverified` on loss of evidence or `violated` on established contradiction |
| `unverified` | Enforcement is not established | Becomes `verified` only on established enforcement, or `violated` on established violation |
| `violated` | The stated boundary is established as violated | May become `verified` only after enforcement is re-established |

A new recording job MUST satisfy its own and inherited constraints for covered processing and storage. A job request MUST NOT relocate an existing session. Observed location, asset origin, a storage bucket or a routing preference alone MUST NOT prove enforcement of another boundary. Claims identify activity, direction, service boundary, mechanism and authoritative evidence. Fake evidence alone is insufficient.

*Example:* An allowed storage region does not establish the location of a recording worker or of temporary fallback copies.

### 11.8 End-to-End Encryption

Version 1 is discovered for the session and media/data scope. Discovery identifies supported content, direction, sources, decrypting endpoints, key responsibility, exclusions and compatibility with enabled features, or unknown.

**configureEncryption(session, policy)**

- **Purpose:** Configure the stated end-to-end protection policy.
- **Inputs:** Session and policy identifying protected content, direction, source scope, decrypting endpoints, key responsibility, exclusions and whether protection is required or preferred.
- **Preconditions:** The caller has authority. Required protection and the complete feature combination are verified before affected content flows or another dependent effect occurs.
- **Postconditions:** The result reports `effective`, `not-effective`, or `unknown` for the stated coverage. Required protection is claimed effective through recovery, replacement and other enabled features only while its coverage remains verified. An unmet preference is reported separately and is not represented as effective.
- **Failures:** The shared failure table in Section 11.4 applies. The table below applies in addition to Section 9.
- **Events:** Changes in protection and loss of verification are observable with their affected scope.
- **Ownership:** The application or designated key authority owns supplied keys. Configuration does not implicitly transfer ownership.
- **Cancellation:** Section 8.5 applies. Known and unknown configuration effects are reported under Section 9.

| Condition | Result |
|---|---|
| Required protection unsupported, incompatible or unverifiable | Reject before the affected effect |
| Preferred protection not selected or not established | Report not selected or unknown under Section 8.8; do not claim effective protection |
| Required protection loses verification | Promptly report loss and stop initiating or forwarding affected activity within implementation control; do not claim remote activity stopped without evidence |
| Failure after a change begins | Report established and unknown effects under Section 9 |

| State | Meaning | Transition |
|---|---|---|
| `effective` | Protection is established for the identified coverage | Becomes `unknown` on loss of verification or `not-effective` on established loss |
| `not-effective` | Protection is established as not effective for that coverage | Becomes `effective` only when established |
| `unknown` | Current protection is not established | Becomes `effective` or `not-effective` only when established |

An encryption claim MUST identify protected media or data, direction, source scope, endpoints able to decrypt, key responsibility and exclusions. Transport encryption MUST NOT be represented as end-to-end encryption. Keys remain within their authorized boundary and are excluded from public state and default diagnostics. Cleanup MUST NOT claim destruction of remote keys without evidence.

*Rationale:* Recovery or recording can change the decrypting endpoints; the original configuration alone does not establish continued coverage.

### 11.9 Server Administration

Version 1 discovery and operations are trusted-server-scoped. Each operation requires its corresponding server authority. Client credentials do not confer that authority.

**createConference(server, options)**

- **Purpose:** Provision a conference.
- **Inputs:** Trusted server and provisioning options.
- **Preconditions:** The caller has provisioning authority and required options are enforceable.
- **Postconditions:** Success returns a server-owned reference to an established active conference.
- **Failures:** The shared failure table in Section 11.4 applies. Section 9 applies. Unknown provisioning outcome MUST NOT be automatically retried without an idempotency guarantee.
- **Events:** Established conference state changes are observable to authorized observers.
- **Ownership:** The trusted server owns the conference reference and its administration scope.
- **Cancellation:** Section 8.5 applies. Known surviving resources and unknown remote effects are reported under Section 9.

**endConference(server, conference)**

- **Purpose:** End the identified conference for all participants.
- **Inputs:** Trusted server and conference reference.
- **Preconditions:** The caller has authority to end that conference. A repeated end is eligible.
- **Postconditions:** The request affects the identified conference. Repeated end is safe. Local request completion does not prove remote termination without evidence.
- **Failures:** The shared failure table in Section 11.4 applies. Section 9 applies. Known and unknown remote effects are reported.
- **Events:** An established end changes conference state to `ended`. Participant session termination is observed where established.
- **Ownership:** The server retains ownership of the conference reference. The operation does not acquire participant-owned resources.
- **Cancellation:** Section 8.5 applies. A cancelled request does not imply that a submitted remote end did not occur.

**removeParticipant(server, conference, participant)**

- **Purpose:** Remove the selected participant presence from the identified conference.
- **Inputs:** Trusted server, conference and participant presence reference.
- **Preconditions:** The caller has removal authority. The reference identifies the intended presence in that conference.
- **Postconditions:** Success identifies the established removal effect. It does not claim an unobserved remote effect.
- **Failures:** The shared failure table in Section 11.4 applies. Invalid or stale references, denied authority and other established conditions use Section 9. Known and unknown remote effects are reported.
- **Events:** Departure is observed where established under Sections 8.2 and 8.3.
- **Ownership:** Removal does not transfer ownership of the participant's resources.
- **Cancellation:** Section 8.5 applies. A submitted removal may have an unknown remote effect.

**mintCredential(server, claims)**

- **Purpose:** Issue a credential for the requested authorized claims.
- **Inputs:** Trusted server and claims identifying intended audience, subject, destination and purpose as applicable.
- **Preconditions:** The caller has issuance authority for the claims.
- **Postconditions:** Success returns the credential only through its authorized secret channel to the authorized recipient.
- **Failures:** The shared failure table in Section 11.4 applies. Invalid claims, denied authority and other established conditions use Section 9. Partial or unknown issuance is reported without disclosing credential material.
- **Events:** Public observations and diagnostics MUST NOT contain the credential.
- **Ownership:** Issuance does not transfer administrative authority to a client credential.
- **Cancellation:** Section 8.5 applies. A late result MUST NOT be delivered as success after cancellation wins.

| Conference state | Trigger | Next |
|---|---|---|
| `active` | End established | `ended` |
| `ended` | Repeated end | `ended` |

`ended` is terminal. Administrative secrets remain on the trusted server. Credentials are confined to authorized recipients and excluded from public observations and diagnostics.

### 11.10 Devices and Statistics

Version 1 discovery is client-scoped for devices and session- or resource-scoped for statistics. Device discovery distinguishes enumeration and selection, media kind, target scope and permission prerequisites. Discovery itself does not prompt.

**listDevices(client, kind)**

- **Purpose:** List observable devices of the requested kind.
- **Inputs:** Client and device kind.
- **Preconditions:** The client is eligible and the caller has authority to observe the requested scope.
- **Postconditions:** The result identifies scope, completeness and permission-dependent unknowns. Restricted information MUST NOT be represented as a complete empty physical inventory. Redacted or unknown labels remain distinct from known labels.
- **Failures:** The shared failure table in Section 11.4 applies. Invalid kind, unsupported enumeration, denied authority and other established conditions use Section 9.
- **Events:** Established inventory and availability changes are observable within declared coverage. Unobservable changes remain unknown.
- **Ownership:** Listing acquires no device or source.
- **Cancellation:** Section 8.5 applies to asynchronous enumeration.

**selectDevice(source, device)**

- **Purpose:** Select a device for the identified source.
- **Inputs:** Source and device reference identifying kind and scope.
- **Preconditions:** The caller has source-owner authority and applicable platform consent. Selection itself does not grant consent.
- **Postconditions:** Success means selection is established. Selection does not establish production or playback. Additional capture or publication effects require Section 7.1 disclosure and acceptance.
- **Failures:** The shared failure table in Section 11.4 applies. Invalid or unavailable devices, unsupported selection, denied authority or consent and other established conditions use Section 9. Partial or unknown effects are reported under Section 9.
- **Events:** Established selection changes and device loss are observable for the affected scope.
- **Ownership:** Selection does not transfer source or device ownership.
- **Cancellation:** Section 8.5 applies, including reporting of unknown effects and unresolved release.

| Device observation | Meaning | Change |
|---|---|---|
| `available` | Availability is established for the declared scope | Becomes `unavailable` or `unknown` when established |
| `unavailable` | Unavailability is established for the declared scope | Becomes `available` or `unknown` when established |
| `unknown` | Current availability is not established | Becomes `available` or `unavailable` only when established |

**getStatistics(resource)**

- **Purpose:** Obtain measurements for the identified resource.
- **Inputs:** Resource and any supported measurement selection.
- **Preconditions:** The resource is in scope and the caller is authorized to observe it.
- **Postconditions:** The result identifies measurement scope, interval, units and known or unknown values. It identifies collection time or unknown and preserves the original time of a cached sample. Unavailable measurements are not zero.
- **Failures:** The shared failure table in Section 11.4 applies. Invalid target, unsupported measurement, denied authority and other established conditions use Section 9.
- **Events:** This operation returns a sample; it does not imply a continuing stream or a fresh network probe.
- **Ownership:** Measurement acquires no device or source.
- **Cancellation:** Section 8.5 applies to asynchronous sampling.

Device identifiers and statistics can reveal identity or activity. They require appropriate access control. Device identifier scope MUST be disclosed. An identifier MUST NOT be represented as stable across scopes without evidence.

### 11.11 Credential Renewal

Version 1 discovery is scoped to provider, resource and phase. Discovery identifies who supplies a replacement, whether replacement is supported, its applicable acknowledgment and the known effect of expiry or revocation, or unknown.

**renewCredential(target, credential)**

- **Purpose:** Replace the credential for the identified target and phase.
- **Inputs:** Target and credential identifying intended provider, audience, subject, destination and purpose.
- **Preconditions:** The caller is authorized to provide a replacement. A known scope mismatch fails before submission.
- **Postconditions:** The result reports `applied` only after the applicable acknowledgment is established. Otherwise it reports `deferred`, `unsupported`, `failed` or `unknown` as established. Cached replacement MUST NOT be reported as live reauthorization without evidence.
- **Failures:** The shared failure table in Section 11.4 applies. The table below applies in addition to Section 9.
- **Events:** Acquisition or renewal failure and later established application are observable without credential material. Failure alone does not establish revocation or disconnection.
- **Ownership:** The credential remains owned by its authorized issuer or application. Renewal does not grant administration.
- **Cancellation:** Section 8.5 applies. A cancelled request or a request overtaken by another renewal MUST NOT install its late credential over the newer credential.

| Condition | Result |
|---|---|
| Known audience, subject, destination, purpose or provider mismatch | Reject before submission |
| Replacement unsupported for target or phase | `unsupported` before replacement effects |
| Acquisition fails | Report failure distinctly from rejection of a supplied credential |
| Application not yet established | Report deferred or unknown as established; do not report applied |
| Renewal fails | Report failure; preserve independently established old credential validity and applicability, or report them unknown |

| Outcome | Meaning | Later observation |
|---|---|---|
| `applied` | Applicable acknowledgment is established | Later expiry or revocation is independently observed |
| `deferred` | Application is pending at a stated boundary | `applied`, `failed` or unknown when established |
| `unsupported` | Replacement is unsupported in the stated scope and phase | A later capability change is independently observed |
| `failed` | Replacement work failed | Old credential validity and session state remain independently established or unknown |
| `unknown` | Application outcome is not established | A later established outcome may refine it |

Credential material is confined to the authorized channel and excluded from public state and diagnostics. Expiry of an old credential MUST NOT by itself invalidate a current pending replacement. Renewal failure MUST NOT itself be reported as revocation or disconnection.

## 12. Security Considerations

### 12.1 Credentials, secrets and diagnostics

A credential can be misused if disclosed, applied to the wrong destination, or treated as authority beyond its grant. The implementation MUST protect credentials in transit and storage. The implementation MUST use credentials only within their known audience, subject, purpose and destination. A known mismatch MUST fail before submission. Possession of a discovery result MUST NOT be treated as authorization. The trusted server holds provider signing and management secrets. Client credentials MUST NOT confer trusted-server authority. Privileged administration requires the corresponding server authority.

The following exclusions apply to credentials, secret-bearing URLs and headers, secret key material, provider signing and management secrets, raw provider payloads and stacks. These materials MUST NOT appear in public results, snapshots or events, except through the explicitly authorized diagnostic access described below. They MUST NOT appear in default logs, error rendering or standard serialization. They MUST NOT appear in reflective enumeration where available, extension diagnostics or remote error representations. Credential delivery to its intended authorized recipient is an exception confined to that authorized channel. Explicitly authorized unrestricted diagnostic access is confined to its intended authorized boundary. A diagnostic accessor MUST NOT expose these materials without an explicit application request. Provider codes and messages MUST be sanitized for their specific fields before exposure. Diagnostic formatting MUST handle cycles and non-error values without exposing unrestricted contents.

Logs and diagnostic data can disclose relationships and activity even without secrets. Implementations SHOULD restrict access to logs and retain them only as needed for their authorized purpose. Applications remain responsible for access to and retention of diagnostics they receive.

### 12.2 Consent, identity and references

Capture, publication, recording and use of an application-owned source each require the applicable explicit application request and authority. Required platform or user consent MUST precede protected use. Pending or denied consent MUST NOT be treated as permission. Discovery MUST NOT prompt for consent. Join alone MUST NOT capture or publish. Explicitly requested effects bundled with join remain subject to Section 7.1 disclosure and consent. Capture permission does not imply permission to publish or record. Mute does not guarantee capture release or exclusion from a recording.

*Example:* A user who permits camera capture has not thereby authorized the application to publish that camera or record it.

An attacker can select a familiar display name or present a misleading address, provider ID or caller ID. An unverified value of any of these MUST NOT be represented as authenticated application identity. Authentication of an application identity requires evidence from the relevant authority. A participant is an endpoint presence, not necessarily a person. The identity and incomplete-roster rules of Section 8.2 prevent omission from being presented as departure and prevent a delayed observation from changing a later presence. A stale, foreign, terminal or unauthorized reference MUST NOT select or affect another resource, including in a later session. A source owner's explicit grant is required for another owner to use the source.

Roster membership, metadata, device identifiers, call destinations and activity can reveal sensitive relationships. Access to roster and metadata MUST be controlled according to the applicable authority. Exposed list scope and completeness MUST be truthful.

### 12.3 Recording and required protection

Recording requires an explicit request, recording authority and applicable participant notice and consent before recording begins. Pending or denied consent is not permission. The Recording profile identifies the notice and consent mechanism and participants covered. Recording storage and playback require access control and retention appropriate to their sensitivity. Local mute does not establish that a provider recording excludes the participant's content. [RFC7866] describes security and notification considerations for a distinct recording protocol; it does not define this profile's behavior.

An end-to-end protection claim MUST identify the protected content, direction, source scope, decrypting endpoints, key responsibility and exclusions. Transport encryption alone MUST NOT be represented as end-to-end encryption. Keys MUST remain within their authorized boundary. A required security or placement condition that is unknown, unsupported or unverifiable MUST fail before dependent effects. When verification is lost, that loss MUST be observable. The implementation MUST prevent further implementation-controlled affected activity outside the required boundary. A claim that a provider service enforces a boundary requires current authoritative evidence. That evidence MUST cover the claimed activity, direction, service boundary and mechanism. Configuration acceptance, observed location and simulated provider behavior do not, by themselves, establish enforcement.

*Example:* Evidence that a recording is stored in an allowed region does not establish that recording processing or media transit remained in that region.

### 12.4 Resource exhaustion and extensions

Requests, listeners, capture, recordings and reconnection consume finite local and provider resources. Implementations SHOULD apply appropriate access controls and resource limits. Quota and availability outcomes MUST be reported accurately under Section 9. Cancellation and cleanup follow the finite settlement and observation rules in Sections 8.5 and 8.6. Local settlement does not guarantee immediate remote termination.

Provider extensions introduce additional inputs and authority boundaries. An extension MUST use its provider-qualified namespace and documented version. An extension MUST validate requests as required by Section 10. An extension MUST NOT alter core operation semantics or confer authority. Core security, error, ownership and cleanup requirements continue to apply to extensions. Unknown, unsupported and wrong-boundary options are rejected before dependent effects under Section 8.8.

Secrets and protected media can still exist at authorized endpoints or providers within the stated trust boundary. SIP [RFC3261] and JSEP [RFC8829] are useful background for deployments using those technologies; this specification does not require either as a wire protocol.

## 13. Privacy Considerations

Participant identity, presence, device identifiers, messages, statistics, recording metadata and call destinations can reveal people, relationships and behavior. Applications and implementations SHOULD collect and expose only the data needed for the authorized purpose. Access to roster and metadata is controlled as specified in Section 12.2. Lists MUST state scope and completeness. Unknown identity MUST remain distinguishable from an established identity. Omission from an incomplete roster MUST NOT be presented as a person's absence.

Screen capture can include unrelated private information. Applicable platform consent governs selection. The selected scope and subsequent changes MUST be represented truthfully under Section 11.4.

Recording and playback can disclose content beyond the live session. The notice, consent, access and retention requirements in Sections 11.6 and 12.3 apply. Session termination or local release MUST NOT be represented as proof of deletion of remote recordings, provider logs or copies without evidence. Applications and providers remain responsible for notice, retention, access and deletion policies within their authority.

## 14. Conformance

### 14.1 Claims and applicability

A core or profile claim MUST identify the specification revision, published artifacts, actual implementation, binding and environment. The claim MUST identify its operations and supported scope, profile versions, dependencies and evidence. Runtime evidence MUST identify the actual implementation and environment tested. Every applicable normative obligation MUST be accounted for.

| Condition | Applicability or result |
|---|---|
| An operation is implemented or advertised within the claim's scope | Its applicable core and profile requirements apply even without a separate profile claim. |
| A core media operation is supported | Its core postconditions apply for each advertised media kind and source scope. |
| An optional positive behavior is unsupported | Its positive assertion is not applicable; applicable negative, security, ownership and cleanup assertions remain mandatory. |
| A profile is claimed | Its defining operations, requirements and declared dependencies are mandatory; an unsupported defining operation fails the profile. |
| A profile has no applicable defining behavior | It cannot pass. |

Each exclusion MUST identify the assertion, reason and evidence. A missing fixture or missing evidence MUST NOT be used as an exclusion.

### 14.2 Verdicts and evidence

Each assertion MUST be reported as `pass`, `fail`, `incomplete`, or `not-applicable`. A not-run assertion or an assertion with unknown evidence is `incomplete` when applicable. An assertion is `not-applicable` only when its applicability condition is not met and the exclusion is justified.

| Claim verdict | Condition |
|---|---|
| `fail` | At least one applicable assertion fails, including a required defining operation that is unsupported. |
| `incomplete` | No applicable assertion fails, but an applicable assertion, required positive behavior, justified applicability determination or required evidence is missing or unknown. |
| `pass` | All applicable assertions pass, required positive behavior exists, exclusions are justified, and current external evidence supports every exact claimed boundary. |
| `not-applicable` | The claim's scope does not apply; this verdict MUST NOT be used to make a profile without applicable defining behavior pass. |

Runtime behavior MUST be tested through public interfaces. Controlled peers, provider instrumentation and independent service evidence MAY provide test inputs or evidence. Internal implementation machinery is not a conformance requirement. A fake result MUST NOT be represented as a real-provider result. A plan or fake-only pass MUST NOT be represented as provider execution.

External enforcement evidence MUST identify the activity, direction, service boundary, mechanism and authority.

The claim MUST identify the evidence source, applicable version and validity period. Evidence is current only within its stated validity period and while not withdrawn or contradicted. An evaluator MUST mark the applicable assertion incomplete when evidence is missing, expired or contradicted; established nonconformance fails it. Withdrawn or contradicted evidence invalidates verification. Fake evidence alone cannot establish a provider service boundary. A release claim MUST identify the published artifacts and revision. It MUST audit the inventory and known evidence gaps. It MUST demonstrate that published client interfaces do not expose trusted-server secrets or unauthorized administration.

*Example:* A controlled peer can verify suppression of source content at that peer, but a fake provider cannot establish that a real provider processes media only within a claimed region.

### 14.3 Inventory and suite

The OpenRTC Foundation MUST publish and maintain a versioned inventory mapping each independently testable obligation in this RFC to its clause. Each entry MUST have a stable identifier.

The inventory MUST include at least the following tests, with stable identifiers and public or independently evidenced oracles:

| Stimulus and observable oracle |
|---|
| Exercise every mandatory core and claimed-profile operation positively; test unsupported optional media and reject a vacuous profile claim. |
| Dismiss selection and race accepted operation cancellation; distinguish reasons and verify no late success. |
| Inject coupled effects and late acquisition with an inseparable release effect; verify disclosure and acceptance before effects, and unresolved scope without a false release if undisclosed effects prevent safe cleanup. |
| Race observer registration and change; inject local delivery loss and verify indication, fresh snapshot and gap-free subsequent handoff. |
| Inject failed release, dispose, inspect the retained result and a later outcome within its declared retention period. |
| Inject remote offers adding a media kind; verify default whole rejection, selected exclusion and fail-before-setup for an unenforceable flow. |
| Send to an offline recipient, reconnect and verify no queued delayed delivery. |
| With collision policy unknown and an existing presence, verify no automatic same-identity retry. |
| Withhold an application-controlled incoming transfer through its deadline; verify rejection without execution, and pre-setup failure for unenforceable policy. |
| Withhold required recording consent and inspect the service for no job or content; verify an authorized independent job survives local leave and remains stoppable. |
| Audit authoritative evidence for encryption content, direction, endpoints, key responsibility and boundary; withdraw or contradict it and verify observable loss and local containment. |
| Request incompatible profile or extension versions and foreign namespaces; verify failure before acquisition or effects, then exercise a supported version. |
| Replace a muted publication with a disabled source; verify both intents, source ownership and pre-effect rejection of unverified required identity continuity. |
| Establish local and remote hold, resume locally, and verify remote hold and unknown states remain independent. |
| Fail and cancel digit submission after a prefix; verify authorized fallback order, no duplicate digit and known or unknown prefix and methods. |
| Delay a credential renewal until a later one is acknowledged; verify the stale completion cannot replace it. |
| Audit a claim with seeded missing obligations, unrun evidence, fake-only provider evidence and invalid exclusions; verify incomplete or failed verdict and affected clauses. |

Each assertion is applicable according to its claim and capability; applicable negative, security and cleanup checks remain required. Missing fixtures or service evidence are incomplete, not passes. Each entry MUST identify its applicable scope and evidence method. The inventory MUST distinguish release-artifact evidence from runtime evidence. The inventory MUST account for obligations not executable by the suite.

Each claim MUST identify the inventory revision used. Applicability follows Section 14.1 and the operation table in Section 7.0. An evaluator MUST apply the identified inventory to the claimed scope, record each applicable assertion and its evidence, and report missing fixtures, evidence or unexecuted assertions as incomplete.

Where applicable, the inventory MUST cover stale observations after termination, identifier reuse and incomplete rosters. It MUST cover independent suppression intents, partial effects and failed cleanup. It MUST cover loss of required protection and secrets on all public surfaces. Each applicable assertion MUST be executed or reported not-run or unknown. Missing observation of an applicable surface is not a pass.

## 15. IANA Considerations

This document requests no IANA action.


## 16. References

### 16.1 Normative References

[RFC2119] Bradner, S., “Key words for use in RFCs to Indicate Requirement Levels”, BCP 14, RFC 2119, March 1997, https://www.rfc-editor.org/rfc/rfc2119.

[RFC8174] Leiba, B., “Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words”, BCP 14, RFC 8174, May 2017, https://www.rfc-editor.org/rfc/rfc8174.

### 16.2 Informative References

[RFC3261] Rosenberg, J., et al., “SIP: Session Initiation Protocol”, RFC 3261, June 2002, https://www.rfc-editor.org/rfc/rfc3261.

[RFC7866] Portman, L., et al., “Session Recording Protocol”, RFC 7866, May 2016, https://www.rfc-editor.org/rfc/rfc7866.

[RFC8829] Uberti, J., Jennings, C., and E. Rescorla, “JavaScript Session Establishment Protocol (JSEP)”, RFC 8829, January 2021, https://www.rfc-editor.org/rfc/rfc8829.

## Appendix A. Interface Definition

### A.0 Notation

This language-neutral IDL specifies operation and type shapes equivalent to the contract in Sections 4–11; the body governs behavior and Section 9 governs failures. A binding supplies equivalent operations and representations without adopting these names, serialization, or class structure. `record` fields are required unless marked `optional`; omission means absent, not unknown. `A | B` is a tagged union; bare words in a union are literal tags; `optional<T>` permits omission. `list<T>` is ordered, `set<T>` is unordered, and `map<K,V>` has unique keys. `string`, `bool`, `bytes`, `uint`, `number`, `duration`, and `timestamp` have their usual mathematical meanings; `void` has no value. Identifiers and references are opaque and scoped as in Section 6. An `async` operation settles once and supports the binding's cancellation mechanism. A `stream<T>` is a removable observation, not an implementation architecture. Values labeled `unknown` or `not-applicable` are distinct from omission and from each other.

All operations below may fail with any of the ten Section 9 core codes when its condition applies; profiles may additionally return a namespaced code with a core code. The following shorthand identifies codes typically returned where their conditions apply; it is informative and is not an exhaustive per-operation failure list: `fails E` = `invalid-argument | invalid-state | unsupported | permission-denied | authentication-failed | cancelled | timeout | unavailable | internal | unknown`. Section 9 determines the applicable code and outcome; a listed code does not imply that every condition is possible for every invocation. Operations inherit the validation, disclosure, consent, concurrency, cancellation, and cleanup rules of Sections 7–9 and 11.

### A.1 Core types

```text
enum CoreCode { invalid-argument, invalid-state, unsupported, permission-denied,
  authentication-failed, cancelled, timeout, unavailable, internal, unknown }
enum Reason { credential-acquisition-failed, credential-expired, credential-revoked,
  authentication-failed, authorization-denied, operation-conflict, resource-not-found,
  resource-conflict, quota-or-rate-limit, network-failure, service-unavailable,
  media-failure, protocol-failure, cleanup-failure, selection-dismissed, unknown }
enum Outcome { none, partial, unknown }
enum Assessment { supported, unsupported, unknown }
enum RejectionAssessment { unsupported, unknown, pending }
enum Certainty { guaranteed, possible, unknown }
enum Completeness { complete, incomplete, unknown }
enum ClientState { ready, disposed }
enum SessionState { connecting, connected, reconnecting, disconnected }
enum DisconnectReason { left, removed, ended, failed, unknown }
enum PresenceState { present, departed }
enum SourceState { live, stopped }
enum PublicationState { active, ended }
enum MediaKind { audio, video, screen }
enum Truth { true, false, unknown, not-applicable }
enum IdentityStatus { authenticated, unauthenticated, unknown, not-applicable }
enum Continuity { established, unknown, not-applicable }
enum CleanupState { pending, failed, unknown, released }
enum Selection { selected, not-selected, unknown }

type Resource = Server | Client | Session | Participant | LocalSource | Publication |
  RemotePublication | Subscription | Call | Grant | Recording | Conference |
  Device | ReceiveSource | RenderAttachment;
record Scope { owner: Resource; description: string }
record Coverage { scope: Scope; completeness: Completeness }
record ResourceSet { scope: Scope; description: string }
type EffectTarget = Resource | ResourceSet;
record Effect { target: EffectTarget; action: string; certainty: Certainty }
record EffectAcceptance { target: EffectTarget; action: string; certainty: Certainty }
record Resolution { mechanism: string; evaluationBoundary: string }
record Condition { name: string; scope: Scope; parameters: map<string, string> }
record RejectedCondition { condition: Condition; assessment: RejectionAssessment;
  resolution: optional<Resolution> }
record RejectedOption { path: string; operation: string;
  problem: invalid-value-or-shape | unsupported-semantics }
record FailureDetails { knownEffects: list<Effect>; unknownEffects: list<Effect>;
  survivingResources: list<Resource>; unstartedSteps: list<string>;
  rejectedOptions: list<RejectedOption>; rejectedConditions: list<RejectedCondition>;
  cleanupFailures: list<CleanupObservation> }
type ProfileQualifiedReason = string; // namespaced as specified in Section 9
record Error { code: CoreCode; message: string; providerCode: optional<string>;
  profileCode: optional<string>; reasons: set<Reason | ProfileQualifiedReason>; outcome: Outcome;
  details: FailureDetails; correlation: optional<string> }

record Client { id: string; state: ClientState }
record Session { id: string; owner: Client; state: SessionState;
  disconnectReason: DisconnectReason | not-applicable }
record Identity { value: string | absent | unknown | not-applicable;
  status: IdentityStatus }
record Participant { id: string; session: Session; state: PresenceState;
  isLocal: bool; applicationIdentity: Identity; providerIdentity: Identity;
  endpointIdentity: Identity; continuity: Continuity; possiblePriorPresence: Participant | unknown | not-applicable }
record LocalSource { id: string; owner: Resource; kind: MediaKind; state: SourceState;
  disabled: Truth; authorizationScope: Scope }
record Publication { id: string; owner: Session | Call; source: LocalSource;
  state: PublicationState; muted: Truth; setMutedIntent: Truth; setSendingMuteIntent: Truth }
record RemotePublication { id: string; session: Session; participant: Participant;
  kind: MediaKind; state: PublicationState; scope: Scope }
record ReceiveSource { id: string; session: Session; scope: Scope;
  composition: single-publication | aggregate | unknown;
  contributors: CoveredRemotePublications | unknown }
record CoveredLocalSources { coverage: Coverage; items: list<LocalSource> }
record CoveredParticipants { coverage: Coverage; items: list<Participant> }
record CoveredPublications { coverage: Coverage; items: list<Publication> }
record CoveredRemotePublications { coverage: Coverage; items: list<RemotePublication> }
record CoveredReceiveSources { coverage: Coverage; items: list<ReceiveSource> }
record Snapshot { scope: Scope; session: Session; sources: CoveredLocalSources;
  participants: CoveredParticipants; publications: CoveredPublications;
  remotePublications: CoveredRemotePublications; receiveSources: CoveredReceiveSources }
record CleanupObservation { target: EffectTarget; state: CleanupState;
  failure: optional<Error>; changes: optional<stream<CleanupChange>> }
record CleanupChange { target: EffectTarget; state: CleanupState; failure: optional<Error> }
record ProfileVersion { name: string; version: string; assessment: Assessment }
record Constraint { name: string; value: string | unknown; scope: Scope }
record CapabilityRequest { operations: list<string>; kinds: list<MediaKind>;
  scope: Scope; conditions: list<Condition>; profiles: list<ProfileVersion> }
record Capability { operation: string; kind: MediaKind | not-applicable;
  scope: Scope; assessment: Assessment; conditions: list<Condition>;
  constraints: list<Constraint>; coupledEffects: list<Effect> }
record CapabilityAssessment { coverage: Coverage; capabilities: list<Capability>;
  combined: Assessment; rejected: list<RejectedCondition>;
  profiles: list<ProfileVersion>; changeNotification: string }
record ProviderOption { namespace: string; schemaVersion: string; key: string; value: bytes }
record Option { path: string; value: bytes; requirement: required | preference }
record Options { portable: list<Option>; provider: list<ProviderOption>;
  requiredConditions: list<Condition>; acceptedEffects: list<EffectAcceptance> }
record PreferenceResult { path: string; selection: Selection; reason: optional<string> }
record OperationOptions { conditions: list<Condition>; acceptedEffects: list<EffectAcceptance>;
  deadline: optional<duration> }
record Destination { provider: string; address: string }
record Credential { secret: bytes; audience: string | unknown; subject: string | unknown;
  purpose: string | unknown; destination: Destination | unknown }
type CredentialProvider = async (Destination) -> Credential fails E;
record JoinOptions { options: Options; credentialProvider: CredentialProvider;
  creation: existing-only | creation-allowed; bundledCapture: list<MediaKind>;
  bundledPublish: bool }
record JoinResult { session: Session; sources: list<LocalSource>;
  publications: list<Publication>; preferences: list<PreferenceResult> }
record PublishResult { publications: list<Publication> }
record ReleaseResult { unresolved: list<CleanupObservation> }
record SendingResult { establishedEffects: list<Effect> }
```

A pending rejected condition includes `resolution`; the other assessments omit it unless a resolution is known. A `Credential` is carried only through an authorized secret-bearing channel. `RecipientScope` represents the explicit direct or broadcast scope defined in Section 11.5; its membership and delivery boundary are disclosed there. An `Error` may have profile-qualified reasons in addition to `Reason`; its correlation value is present only when established. An observed population list without its own Coverage inherits Coverage from its enclosing value only when that value explicitly supplies Coverage. Scope alone does not establish completeness. Observed population lists identify coverage and completeness under Section 4.

```text
record StateChange { state: SessionState; reason: DisconnectReason | not-applicable }
record ParticipantChange { participant: Participant }
record PublicationChange { publication: Publication | RemotePublication }
record SourceChange { source: LocalSource }
record CapabilityChange { assessment: CapabilityAssessment }
record Loss { scope: Scope; resynchronizationRequired: bool }
record UnsolicitedError { error: Error }
record ReceiveSourceChange { source: ReceiveSource }
type Change = StateChange | ParticipantChange | PublicationChange | SourceChange |
  ReceiveSourceChange | CapabilityChange | CleanupChange | UnsolicitedError;
record Update { scope: Scope; changes: list<Change> }
type SessionEvent = Update | Loss | Snapshot;
record Observer { initial: Snapshot; events: stream<SessionEvent> }
```

The publication mute fields represent the independent intents and their effective union as specified in Section 7.4. The Section 8.1 setter transitions change only the named intent; an unchanged effective union does not require a new effective-mute event. An `Update` represents related facts of one indivisible observation together. A `Loss` and subsequent snapshot obey Section 8.3. `Snapshot` in an event is a resynchronization, not a second application of preceding changes.

### A.2 Core operations

```text
async createClient(options: Options) -> Client fails E
async join(client: Client, destination: Destination, options: JoinOptions) -> JoinResult fails E
async leave(session: Session) -> ReleaseResult fails E
async dispose(client: Client) -> ReleaseResult fails E
async capabilities(scope: Scope, request: CapabilityRequest) -> CapabilityAssessment fails E
async observe(session: Session) -> Observer fails E
async snapshot(session: Session) -> Snapshot fails E
async unsubscribe(observer: Observer) -> void fails E
async capture(session: Session, kind: MediaKind, options: Options) -> LocalSource fails E
async publish(session: Session, source: LocalSource, options: OperationOptions) -> PublishResult fails E
async setMuted(publication: Publication, muted: bool, options: OperationOptions) -> void fails E
async setDisabled(source: LocalSource, disabled: bool, options: OperationOptions) -> void fails E
async unpublish(publication: Publication, options: OperationOptions) -> ReleaseResult fails E
async stop(source: LocalSource, options: OperationOptions) -> ReleaseResult fails E
async setSending(publication: Publication, enabled: bool, options: OperationOptions)
  -> SendingResult fails E  // optional convenience
```

Each operation also accepts the binding's cancellation and deadline mechanism where applicable. Cancellation reports whether it won within the declared finite bound measured from receipt of a valid cancellation request; a winning cancellation also settles the cancelled operation within that bound, as specified in Section 8.5. A binding documents how post-disposal cleanup outcomes are inspected and their retention period; an optional observation release is distinct from resource release. The credential callback may be invoked again only as permitted by the applicable renewal contract.

### A.3 Profile types and operations

All profile definitions in Section 11 are version 1. Call hold and transfer are independent overlays even when a binding presents one CallState; unknown hold is represented by the hold observations and, for the answered-call case specified in Section 11.1, the displayed unknown state. The unknown displayed state retains the established answer and is eligible for the answered-call operations and any-nonterminal termination transition. Profile operations use the informative `fails E` shorthand with the profile-code rule in A.0. Profile streams are independently removable through the binding's observation mechanism.

A documented shared observeProfile entry point MAY provide each profile stream.

Screen selection scope changes and remote-update decisions are selectable events. A binding MUST document their event representations, including scope, decision and established added and removed kinds or unknown. It identifies scope and selectable event types, delivers initial state and subsequent changes under Section 8.3, and permits independent removal. It covers incoming calls, call and transfer updates, reception, digits, grants, recording, placement, encryption, devices, renewal and screen selection changes where supported. Call-scoped media uses captureCall or an owner-granted existing source and publishCall; neither requires a Session. A publication returned by publishCall is owned by the Call; a publication returned by publish is owned by the Session. The source retains its existing owner.

```text
enum CallState { dialing, ringing, early-media, active, held, transferring, ended, unknown }
enum CallDirection { incoming, outgoing }
enum SubscriptionState { selected, ended }
enum GrantState { active, revoked }
enum RecordingState { recording, stopped, unknown }
enum PlacementState { verified, unverified, violated }
enum EncryptionState { effective, not-effective, unknown }
enum ConferenceState { active, ended }
enum DeviceState { available, unavailable, unknown }
enum RenewalState { applied, deferred, unsupported, failed, unknown }
enum MessageState { submitted, delivered, failed, unknown }
enum ReplacementIdentity { preserved, changed, unknown }
record Call { id: string; owner: Client; direction: CallDirection; state: CallState;
  localHold: Truth; remoteHold: Truth; transferActive: Truth; callerIdentity: Identity;
  mediaAvailability: string | unknown; terminalReason: string | unknown | not-applicable;
  transferProgress: string | unknown | not-applicable }
enum ReceptionState { inactive, starting, receiving, stopping, failed, unknown }
record ReceptionObservation { scope: Scope; state: ReceptionState; validity: Truth;
  expiry: timestamp | unknown | not-applicable; refreshActive: Truth }
record CallUpdate { call: Call }
record DigitEvent { call: Call; digits: string; readinessScope: Scope }
record DigitSubmission { submittedPrefix: string | unknown;
  methods: list<string> | unknown; status: complete | partial | failed | unknown }
type TransferDestination = Destination | Call;
record TransferResult { acceptance: bool; outcome: completed | failed | unknown;
  calls: list<Call> }
record TransferChange { call: Call; result: TransferResult }
record Subscription { id: string; owner: Session; publication: RemotePublication;
  state: SubscriptionState }
record Grant { id: string; owner: Resource; source: LocalSource; recipient: Resource;
  scope: Scope; state: GrantState }
record ReplacementResult { identity: ReplacementIdentity; publications: list<Publication> }
type RecipientScope = direct { recipients: list<Participant> } |
  broadcast { scope: Scope; membershipBoundary: string };
record Message { sender: Identity; recipients: RecipientScope; payload: bytes;
  state: MessageState }
record RecipientOutcome { recipient: Participant; state: MessageState }
record MessageSubmission { state: MessageState; recipients: list<RecipientOutcome> }
type MessageEvent = Message | Loss;
record MessageObserver { coverage: Coverage; events: stream<MessageEvent> }
enum RecordingOwnership { session-scoped, independent-server-job }
record Recording { id: string; owner: Resource; ownership: RecordingOwnership;
  scope: Scope; state: RecordingState; consentCoverage: Coverage | unknown;
  processingScope: Scope | unknown; storageScope: Scope | unknown }
record PlacementRequirement { scope: Scope; activities: list<string>;
  allowedRegions: set<string>; transit: bool; fallback: bool;
  requirement: required | preference }
record PlacementResult { assessment: Assessment; state: PlacementState; boundary: string | unknown;
  rejected: list<RejectedCondition>; preferences: list<PreferenceResult> }
record EncryptionPolicy { content: list<string>; direction: string; sources: Scope;
  endpoints: list<Identity>; keyResponsibility: string; exclusions: list<string>;
  required: bool }
record EncryptionResult { state: EncryptionState; scope: Scope }
record Conference { id: string; owner: Server; state: ConferenceState }
record Server { id: string }
record CredentialClaims { audience: string; subject: string; purpose: string;
  destination: Destination; claims: map<string, string> }
record Device { id: string; kind: MediaKind; state: DeviceState }
record DeviceList { coverage: Coverage; items: list<Device>; permission: Truth }
record Measurement { name: string; scope: Scope; start: timestamp; end: timestamp;
  collectedAt: timestamp | unknown; unit: string; value: number | unknown }
record Statistics { coverage: Coverage; measurements: list<Measurement> }
record RenewalResult { state: RenewalState }
record ScreenCaptureResult { sources: list<LocalSource>; selectionScope: Scope | unknown;
  audioAssociation: string | unknown | not-applicable }
record ProfileObserver { scope: Scope; initial: list<ProfileEvent>; events: stream<ProfileEvent> }
record PolicyChange { scope: Scope; decision: string; addedKinds: list<MediaKind> | unknown;
  removedKinds: list<MediaKind> | unknown }
record ScreenSelectionChange { scope: Scope; selection: Scope | unknown }
record RenderAttachment { id: string; owner: Session; source: ReceiveSource; target: string }
record SubscriptionChange { subscription: Subscription }
record GrantChange { grant: Grant }
record RecordingChange { recording: Recording }
record PlacementChange { scope: Scope; result: PlacementResult }
record EncryptionChange { session: Session; result: EncryptionResult }
record DeviceChange { device: Device }
record RenewalFailure { target: Resource; error: Error }
type ProfileEvent = CallUpdate | TransferChange | DigitEvent | ReceptionObservation | SubscriptionChange | GrantChange |
  RecordingChange | PlacementChange | EncryptionChange | DeviceChange | RenewalFailure |
  MessageEvent | PolicyChange | ScreenSelectionChange | Loss | UnsolicitedError;
```

```text
async observeProfile(scope: Scope, eventTypes: list<string>) -> ProfileObserver fails E
async captureCall(call: Call, kind: MediaKind, options: Options) -> LocalSource fails E
async publishCall(call: Call, source: LocalSource, options: OperationOptions) -> PublishResult fails E
async dial(client: Client, destination: Destination, options: Options) -> Call fails E
async accept(call: Call, options: OperationOptions) -> void fails E
async reject(call: Call, options: OperationOptions) -> void fails E
async hangup(call: Call, options: OperationOptions) -> ReleaseResult fails E
async setHold(call: Call, held: bool, options: OperationOptions) -> Call fails E
async transfer(call: Call, destination: TransferDestination, options: OperationOptions)
  -> TransferResult fails E  // separately advertised
async sendDigits(call: Call, digits: string, methods: list<string>, options: OperationOptions)
  -> DigitSubmission fails E  // separately advertised
// Incoming calls, CallUpdate and DigitEvent are observable; digit reception is separately advertised.
async startReceiving(client: Client, account: Scope, options: Options) -> void fails E
async stopReceiving(client: Client, account: Scope) -> ReleaseResult fails E
async subscribe(session: Session, publication: RemotePublication, options: OperationOptions)
  -> Subscription fails E
async unsubscribeMedia(subscription: Subscription) -> ReleaseResult fails E
async shareSource(source: LocalSource, recipient: Resource, grantScope: Scope,
  options: OperationOptions) -> Grant fails E
async revokeSourceGrant(grant: Grant, options: OperationOptions) -> void fails E
async replaceSource(publication: Publication, source: LocalSource, requireIdentity: bool,
  options: OperationOptions) -> ReplacementResult fails E
async captureScreen(session: Session, options: Options) -> ScreenCaptureResult fails E
async sendMessage(session: Session, recipients: RecipientScope, payload: bytes,
  options: Options) -> MessageSubmission fails E
async observeMessages(session: Session) -> MessageObserver fails E
async startRecording(session: Session, options: Options) -> Recording fails E
async startIndependentRecording(server: Server, conference: Conference, options: Options)
  -> Recording fails E
async stopRecording(recording: Recording) -> ReleaseResult fails E
async assessPlacement(scope: Scope, requirements: list<PlacementRequirement>)
  -> PlacementResult fails E
async applyPlacement(scope: Scope, requirements: list<PlacementRequirement>,
  options: OperationOptions) -> PlacementResult fails E
async configureEncryption(session: Session, policy: EncryptionPolicy,
  options: OperationOptions) -> EncryptionResult fails E
async createConference(server: Server, options: Options) -> Conference fails E
async endConference(server: Server, conference: Conference) -> ReleaseResult fails E
async removeParticipant(server: Server, conference: Conference, participant: Participant)
  -> void fails E
async mintCredential(server: Server, claims: CredentialClaims) -> Credential fails E
async listDevices(client: Client, kind: MediaKind) -> DeviceList fails E
async selectDevice(source: LocalSource, device: Device, options: OperationOptions)
  -> Device fails E
async getStatistics(resource: Resource) -> Statistics fails E
async renewCredential(target: Resource, credential: Credential) -> RenewalResult fails E
```

*Example:* A message submitted locally can remain `unknown` for a recipient; local submission is not evidence of delivery. A muted publication remains muted after its source is re-enabled.

## Appendix B. Non-normative Implementation Guidance

An implementation can use a native bridge, serialized queues, sequence numbers, receipts, operation correlation keys, cleanup registries and reconciliation workers to provide the observable guarantees. A registry can retain unresolved release observations independently of a disposed session; a late completion handler can release a newly acquired resource without reviving the public session. Per-field configuration receipts and effect ledgers can help diagnostics, provided they do not expose secrets or imply facts that are unknown. A controlled peer and provider fault injection can exercise races and partial outcomes; independent service evidence is needed for claims such as residency or decrypting endpoints.
