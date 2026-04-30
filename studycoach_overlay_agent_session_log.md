# StudyCoach Overlay — AI Coding Agent Session Log

> **Document type:** reconstructed coding-agent build log  
> **Source basis:** macOS overlay feasibility research + the provided `Preflight` coding-agent session log style  
> **Important note:** This is written as a realistic end-to-end build log for the product architecture. It is not a claim that every file already exists in a real repository unless you implement it.

---

## Project Overview

**StudyCoach Overlay** is a native macOS student AI coach that lives as a lightweight, summonable overlay. A student can select text, paste an assignment, or share content into the app. The app then identifies the task, asks targeted clarification questions, applies a teacher-safe honesty policy, and generates learning-focused help such as hints, outlines, rubric maps, quizzes, and next-step study plans.

The product is intentionally **not** a generic chatbot and not just a prompt rewriter. It behaves like a macOS-native study layer:

- stays out of the way until summoned
- captures text through explicit, user-controlled flows
- asks one useful question at a time
- defaults to learning support over answer dumping
- routes sensitive work locally when possible
- routes hosted model calls through a backend broker, not directly from the Mac app
- shows what leaves the device before cloud inference
- keeps academic-integrity controls visible

**MVP Stack**

| Layer | Stack |
|---|---|
| macOS app | Swift, AppKit shell, SwiftUI UI, Observation/Combine-style state |
| Overlay shell | `NSPanel`, `NSStatusItem`, global hotkey controller |
| Capture | `NSPasteboard`, Services/Share extension, optional Accessibility selected text |
| OCR later | ScreenCaptureKit + Vision, deliberately post-MVP |
| Local storage | SwiftData or lightweight JSON store, Keychain for secrets |
| Backend broker | TypeScript + Fastify API gateway |
| LLM routing | provider adapters for OpenAI, Anthropic, Gemini, optional local/on-device lane |
| Testing | XCTest, permission-state manual test matrix, backend unit tests |
| Distribution | direct Developer ID signed + notarized `.app` first, App Store later if compatible |

---

## Phase 0 — Research Translation Into Build Requirements

### Diagnosis: What the Research Changed

The original instinct was to build a simple prompt-enhancer overlay. After reviewing the macOS feasibility report, the product needed a more careful native architecture.

The big conclusions:

1. **This should be a native Mac utility, not a web wrapper.**  
   A browser app cannot reliably behave like a system overlay. The app needs AppKit-level window control, menu bar behavior, global shortcut handling, pasteboard access, and optional Accessibility integration.

2. **AppKit should own the shell; SwiftUI should own the interface.**  
   SwiftUI is faster for UI iteration, but the overlay itself needs `NSPanel`, `NSStatusItem`, window levels, non-activating behavior, and collection behaviors. Those are AppKit-native concerns.

3. **The MVP should be text-first.**  
   Clipboard, paste, Services/Share, and selected-text capture are enough for a strong MVP. Screen OCR is powerful but triggers Screen Recording permission and creates extra privacy friction.

4. **Student data changes the architecture.**  
   Assignment text, rubrics, teacher comments, and school identifiers are sensitive. Hosted model calls should go through a backend broker, not direct API keys inside the app.

5. **Teacher-safe mode is a core feature, not a compliance afterthought.**  
   The app needs explicit help levels: Tutor Only, Coach, and Draft Partner.

### Product Requirements Locked

#### Core MVP

- Menu bar app with optional Dock hiding
- Global shortcut to summon overlay
- Floating non-activating panel
- Paste assignment text into overlay
- Detect clipboard text and offer “Use copied text”
- Optional “capture selected text” using Accessibility permission
- Assignment parser
- Adaptive interview engine
- Quick / Deep / Learning modes
- Tutor Only / Coach / Draft Partner honesty levels
- Backend LLM broker
- Local session history
- Keychain-backed auth/session token storage
- Permission onboarding screen
- Direct notarized distribution path

#### Explicitly Deferred

- Full-screen OCR
- always-on screen watcher
- keyboard event taps
- privileged helper tools
- LaunchDaemon
- school admin dashboard
- classroom policy sync
- multi-device sync
- App Store release
- voice tutoring

### Initial Repo Created

```text
studycoach-overlay/
  macApp/
    StudyCoach.xcodeproj
    StudyCoach/
      App/
      Overlay/
      Capture/
      Interview/
      Policy/
      Routing/
      Security/
      Storage/
      DesignSystem/
      Resources/
    StudyCoachTests/
  backend/
    src/
      server.ts
      routes/
      providers/
      policy/
      schemas/
      telemetry/
    test/
  docs/
    architecture.md
    privacy-model.md
    permission-matrix.md
    agent-session-log.md
```

### Agent Decision Log

| Decision | Chosen Path | Rejected Path | Reason |
|---|---|---|---|
| App framework | Hybrid AppKit + SwiftUI | pure SwiftUI | Overlay shell needs AppKit control |
| First capture path | paste + clipboard + Services | screen OCR | lower permission friction |
| LLM API access | backend broker | API keys in app | protects provider keys and centralizes policy |
| Distribution | direct notarized build | Mac App Store first | overlay utilities face review and sandbox friction |
| Student output | coaching modes | raw answer generation | safer for learning and schools |

---

## Phase 1 — Native macOS Shell

### Goal

Create a small native Mac app that feels like a system utility, not a normal document app.

The agent built:

- app entry point
- menu bar status item
- global summon shortcut placeholder
- non-activating overlay panel
- SwiftUI root view hosted inside AppKit
- overlay state model
- first-run permission checklist

---

### Files Added

```text
macApp/StudyCoach/App/StudyCoachApp.swift
macApp/StudyCoach/App/AppDelegate.swift
macApp/StudyCoach/Overlay/OverlayPanelController.swift
macApp/StudyCoach/Overlay/OverlayWindow.swift
macApp/StudyCoach/Overlay/OverlayRootView.swift
macApp/StudyCoach/Overlay/OverlayViewModel.swift
macApp/StudyCoach/App/StatusBarController.swift
macApp/StudyCoach/App/HotkeyController.swift
macApp/StudyCoach/DesignSystem/CoachTheme.swift
```

---

### `StudyCoachApp.swift`

The SwiftUI app entry point delegates system-level behavior to `AppDelegate`.

```swift
import SwiftUI

@main
struct StudyCoachApp: App {
    @NSApplicationDelegateAdaptor(AppDelegate.self) private var appDelegate

    var body: some Scene {
        Settings {
            SettingsView()
        }
    }
}
```

### `AppDelegate.swift`

The app initializes:

- status bar menu
- overlay panel
- global hotkey controller
- app activation policy

```swift
import AppKit

final class AppDelegate: NSObject, NSApplicationDelegate {
    private var statusBarController: StatusBarController?
    private var overlayController: OverlayPanelController?
    private var hotkeyController: HotkeyController?

    func applicationDidFinishLaunching(_ notification: Notification) {
        NSApp.setActivationPolicy(.accessory)

        let overlayController = OverlayPanelController()
        self.overlayController = overlayController

        self.statusBarController = StatusBarController(
            onToggleOverlay: { overlayController.toggle() },
            onOpenSettings: { NSApp.sendAction(Selector(("showSettingsWindow:")), to: nil, from: nil) },
            onQuit: { NSApp.terminate(nil) }
        )

        self.hotkeyController = HotkeyController {
            overlayController.toggle()
        }
    }
}
```

### Overlay Panel

The key product decision was to use `NSPanel`, not a normal `NSWindow`.

The overlay needs to:

- float above normal windows
- not steal focus unless the student starts typing in it
- appear across Spaces
- feel like a utility panel
- hide when dismissed
- support SwiftUI content

```swift
import AppKit
import SwiftUI

final class OverlayPanelController {
    private let viewModel = OverlayViewModel()
    private lazy var panel: NSPanel = makePanel()

    func toggle() {
        if panel.isVisible {
            panel.orderOut(nil)
        } else {
            show()
        }
    }

    func show() {
        positionPanel()
        panel.makeKeyAndOrderFront(nil)
    }

    private func makePanel() -> NSPanel {
        let rootView = OverlayRootView(viewModel: viewModel)
        let hostingController = NSHostingController(rootView: rootView)

        let panel = NSPanel(
            contentRect: NSRect(x: 0, y: 0, width: 460, height: 560),
            styleMask: [.nonactivatingPanel, .titled, .fullSizeContentView],
            backing: .buffered,
            defer: false
        )

        panel.contentViewController = hostingController
        panel.titleVisibility = .hidden
        panel.titlebarAppearsTransparent = true
        panel.isFloatingPanel = true
        panel.level = .floating
        panel.collectionBehavior = [.canJoinAllSpaces, .transient, .fullScreenAuxiliary]
        panel.hidesOnDeactivate = false
        panel.isReleasedWhenClosed = false
        panel.backgroundColor = .clear
        panel.isOpaque = false

        return panel
    }

    private func positionPanel() {
        guard let screen = NSScreen.main else { return }

        let visible = screen.visibleFrame
        let size = panel.frame.size
        let x = visible.maxX - size.width - 24
        let y = visible.maxY - size.height - 24

        panel.setFrameOrigin(NSPoint(x: x, y: y))
    }
}
```

### Agent Notes

The first version used a normal `WindowGroup`, but it felt like opening an app, not summoning a study layer. Switching to `NSPanel` made the product feel more like Raycast, Spotlight, or a system overlay.

### Verification

| Test | Expected | Result |
|---|---|---|
| Launch app | menu bar icon appears | pass |
| Click status item | overlay opens | pass |
| Toggle overlay twice | opens, then closes | pass |
| Switch desktop Space | panel can join all Spaces | pass |
| Open overlay over Safari | panel floats above normal windows | pass |
| App Dock icon | hidden/accessory behavior | pass |

### Bugs Fixed

- **Panel appeared behind full-screen apps**  
  Added `.fullScreenAuxiliary` to `collectionBehavior`.

- **Panel had ugly title chrome**  
  Added `.fullSizeContentView`, `titleVisibility = .hidden`, and transparent titlebar.

- **Overlay appeared at wrong screen position on external monitor**  
  Added `NSScreen.main.visibleFrame` positioning and later planned cursor-screen detection.

---

## Phase 2 — Overlay UI and Student-Focused Interaction Model

### Diagnosis: What the First UI Got Wrong

The first prototype looked like a normal chatbot panel. That was wrong.

Students do not want another blank AI box. They need the app to understand:

- what class this is for
- what assignment type it is
- what the teacher expects
- whether AI is allowed
- whether they need a hint, outline, explanation, or study plan

### UX Direction

The overlay became a **three-state coach panel**:

1. **Peek State**  
   Small summary: “I found a Biology essay prompt. 2 missing details.”

2. **Coach State**  
   Asks one targeted question at a time.

3. **Session State**  
   Larger view with rubric map, study plan, and follow-up actions.

---

### Files Added / Updated

```text
macApp/StudyCoach/Overlay/OverlayRootView.swift
macApp/StudyCoach/Overlay/CoachHeaderView.swift
macApp/StudyCoach/Overlay/CaptureInputView.swift
macApp/StudyCoach/Overlay/ModeSelectorView.swift
macApp/StudyCoach/Overlay/HonestyLevelSelectorView.swift
macApp/StudyCoach/Overlay/NextQuestionCard.swift
macApp/StudyCoach/Overlay/StudyPlanView.swift
macApp/StudyCoach/DesignSystem/CoachTheme.swift
```

---

### Overlay View Model

```swift
import Foundation
import Observation

@Observable
final class OverlayViewModel {
    var rawInput: String = ""
    var capturedSource: CaptureSource = .manualPaste
    var mode: CoachMode = .quick
    var honestyLevel: HonestyLevel = .tutorOnly
    var phase: OverlayPhase = .empty

    var assignment: AssignmentDraft?
    var nextQuestion: CoachQuestion?
    var studyPlan: StudyPlan?
    var errorMessage: String?

    func useClipboardText() {
        Task {
            do {
                let text = try await CaptureCoordinator.shared.readClipboardText()
                rawInput = text
                phase = .captured
                await analyzeInput()
            } catch {
                errorMessage = error.localizedDescription
            }
        }
    }

    func analyzeInput() async {
        guard rawInput.trimmingCharacters(in: .whitespacesAndNewlines).isEmpty == false else {
            phase = .empty
            return
        }

        phase = .analyzing

        do {
            let draft = try AssignmentParser.parse(rawInput)
            assignment = draft

            if let question = InterviewEngine.nextQuestion(for: draft) {
                nextQuestion = question
                phase = .questioning
            } else {
                phase = .ready
            }
        } catch {
            errorMessage = error.localizedDescription
            phase = .error
        }
    }
}
```

### Core Types

```swift
enum OverlayPhase {
    case empty
    case captured
    case analyzing
    case questioning
    case ready
    case generating
    case result
    case error
}

enum CoachMode: String, CaseIterable, Identifiable {
    case quick
    case deep
    case learning

    var id: String { rawValue }
}

enum HonestyLevel: String, CaseIterable, Identifiable {
    case tutorOnly
    case coach
    case draftPartner

    var id: String { rawValue }
}

enum CaptureSource {
    case manualPaste
    case clipboard
    case accessibilitySelection
    case shareExtension
    case screenshotOCR
}
```

---

### UI Copy Decisions

The agent replaced generic AI wording with student-centered language.

| Generic Copy | Final Copy |
|---|---|
| “Enter prompt” | “Paste an assignment, rubric, or confusing question” |
| “Submit” | “Start coaching” |
| “AI mode” | “Help level” |
| “Generate answer” | “Build study plan” |
| “Output” | “What you can do next” |
| “Error: missing context” | “I need one more detail before I can help responsibly” |

### Mode Design

```swift
extension CoachMode {
    var title: String {
        switch self {
        case .quick: "Quick"
        case .deep: "Deep"
        case .learning: "Learning"
        }
    }

    var subtitle: String {
        switch self {
        case .quick: "1–3 questions, fast next step"
        case .deep: "Rubric map, outline, explanation"
        case .learning: "Hints, quizzes, mastery checks"
        }
    }
}
```

### Honesty Level Design

```swift
extension HonestyLevel {
    var title: String {
        switch self {
        case .tutorOnly: "Tutor only"
        case .coach: "Coach"
        case .draftPartner: "Draft partner"
        }
    }

    var description: String {
        switch self {
        case .tutorOnly:
            "Hints, questions, and explanations. No final answer."
        case .coach:
            "Outlines, scaffolds, partial examples, and feedback."
        case .draftPartner:
            "Exemplar drafts clearly labeled as AI-generated."
        }
    }
}
```

---

### Verification

| Flow | Expected | Result |
|---|---|---|
| Empty overlay | explains what to paste | pass |
| Paste assignment | parser runs | pass |
| Missing teacher AI policy | asks follow-up question | pass |
| Quick mode | smaller question budget | pass |
| Learning mode | avoids final-answer framing | pass |
| Tutor Only selected | final-answer actions hidden | pass |

---

## Phase 3 — Capture Pipeline

### Diagnosis: Why Capture Needed Multiple Paths

No single macOS capture method is reliable enough.

The research pointed to four capture tiers:

1. manual paste
2. clipboard detection
3. Services / Share extension
4. Accessibility selected text

Screen OCR was intentionally deferred.

### Capture Goals

- do not read clipboard constantly without reason
- make capture user-driven
- avoid any keylogging-like behavior
- ask for Accessibility permission only when needed
- treat capture failure as normal, not catastrophic

---

### Files Added

```text
macApp/StudyCoach/Capture/CaptureCoordinator.swift
macApp/StudyCoach/Capture/PasteboardCapture.swift
macApp/StudyCoach/Capture/AccessibilityCapture.swift
macApp/StudyCoach/Capture/CaptureError.swift
macApp/StudyCoach/Capture/CapturePreview.swift
macApp/StudyCoach/Capture/ClipboardWatcher.swift
macApp/StudyCoach/Extensions/ShareExtension/
```

---

### Capture Coordinator

```swift
import Foundation

final class CaptureCoordinator {
    static let shared = CaptureCoordinator()

    private let pasteboardCapture = PasteboardCapture()
    private let accessibilityCapture = AccessibilityCapture()

    private init() {}

    func readClipboardText() async throws -> String {
        try pasteboardCapture.readString()
    }

    func readSelectedText() async throws -> String {
        try accessibilityCapture.readSelectedTextFromFocusedElement()
    }
}
```

### Pasteboard Capture

```swift
import AppKit

struct PasteboardCapture {
    func readString() throws -> String {
        guard let text = NSPasteboard.general.string(forType: .string) else {
            throw CaptureError.noTextAvailable
        }

        let trimmed = text.trimmingCharacters(in: .whitespacesAndNewlines)

        guard trimmed.isEmpty == false else {
            throw CaptureError.noTextAvailable
        }

        return trimmed
    }

    func hasLikelyText() -> Bool {
        NSPasteboard.general.types?.contains(.string) == true
    }
}
```

### Clipboard Watcher

The first version polled the pasteboard every second. That worked, but it felt too invasive.

The agent changed it to:

- only check `changeCount`
- only show a “Use copied text” chip
- never auto-send clipboard contents
- never store clipboard content unless the student clicks the chip

```swift
import AppKit

@Observable
final class ClipboardWatcher {
    private var lastChangeCount = NSPasteboard.general.changeCount
    var hasNewText: Bool = false

    func tick() {
        let pasteboard = NSPasteboard.general
        guard pasteboard.changeCount != lastChangeCount else { return }

        lastChangeCount = pasteboard.changeCount
        hasNewText = pasteboard.types?.contains(.string) == true
    }
}
```

### Accessibility Capture

Accessibility capture was added as an optional enhancement.

```swift
import ApplicationServices
import AppKit

struct AccessibilityCapture {
    func isTrusted(prompt: Bool = false) -> Bool {
        let key = kAXTrustedCheckOptionPrompt.takeUnretainedValue() as String
        let options = [key: prompt] as CFDictionary
        return AXIsProcessTrustedWithOptions(options)
    }

    func readSelectedTextFromFocusedElement() throws -> String {
        guard isTrusted(prompt: false) else {
            throw CaptureError.accessibilityPermissionRequired
        }

        let system = AXUIElementCreateSystemWide()

        var focusedObject: AnyObject?
        let focusedResult = AXUIElementCopyAttributeValue(
            system,
            kAXFocusedUIElementAttribute as CFString,
            &focusedObject
        )

        guard focusedResult == .success,
              let focusedElement = focusedObject else {
            throw CaptureError.noFocusedElement
        }

        var selectedTextObject: AnyObject?
        let selectedResult = AXUIElementCopyAttributeValue(
            focusedElement as! AXUIElement,
            kAXSelectedTextAttribute as CFString,
            &selectedTextObject
        )

        guard selectedResult == .success,
              let selectedText = selectedTextObject as? String,
              selectedText.trimmingCharacters(in: .whitespacesAndNewlines).isEmpty == false else {
            throw CaptureError.noTextSelected
        }

        return selectedText
    }
}
```

### Permission Copy

The agent added plain-language permission messaging:

```text
Selected Text Capture

StudyCoach can read the text you explicitly select in another app.
This helps you send an assignment or confusing question into the coach without copying and pasting.

StudyCoach cannot see everything you type.
It only tries to read the current selected text when you click "Capture selection."
```

### Capture Fallback Behavior

| Situation | Behavior |
|---|---|
| Clipboard has text | show “Use copied text” chip |
| Clipboard has no text | keep manual paste field active |
| Accessibility not trusted | show permission explainer |
| Focused app does not expose text | show “Copy and paste instead” fallback |
| Selection empty | show “Select text first” message |
| OCR needed | show disabled “Screenshot OCR coming later” option |

### Verification

| Test | Expected | Result |
|---|---|---|
| Copy text in Safari | chip appears | pass |
| Click chip | text fills overlay | pass |
| Copy image | no text chip | pass |
| Capture selection without permission | permission screen shown | pass |
| Grant Accessibility | selected text capture works in Notes | pass |
| Capture selected text in PDF viewer | sometimes fails gracefully | pass |
| Capture from Google Docs | fallback copy/paste recommended | expected limitation |

---

## Phase 4 — Assignment Parser and Normalizer

### Goal

Turn messy pasted text into a structured `AssignmentDraft`.

The parser does not need perfect AI reasoning in the first version. It needs enough structure to ask better questions.

---

### Files Added

```text
macApp/StudyCoach/Interview/AssignmentDraft.swift
macApp/StudyCoach/Interview/AssignmentParser.swift
macApp/StudyCoach/Interview/AssignmentSignals.swift
macApp/StudyCoach/Interview/RubricItem.swift
```

### Core Model

```swift
struct AssignmentDraft: Codable, Equatable {
    var rawText: String
    var inferredSubject: Subject?
    var taskType: TaskType
    var deliverable: Deliverable?
    var dueDateText: String?
    var rubricItems: [RubricItem]
    var constraints: [String]
    var missingFields: [MissingField]
    var confidence: Double
}

enum TaskType: String, Codable {
    case mathProblem
    case essay
    case labReport
    case project
    case studyGuide
    case worksheet
    case presentation
    case unknown
}

enum Subject: String, Codable {
    case math
    case science
    case english
    case history
    case worldLanguage
    case computerScience
    case health
    case unknown
}
```

### Parser Implementation

The first parser used deterministic signals:

```swift
enum AssignmentParser {
    static func parse(_ text: String) throws -> AssignmentDraft {
        let normalized = normalize(text)
        let taskType = inferTaskType(normalized)
        let subject = inferSubject(normalized)
        let rubric = extractRubricItems(normalized)
        let constraints = extractConstraints(normalized)
        let dueDate = extractDueDate(normalized)

        let missing = inferMissingFields(
            text: normalized,
            taskType: taskType,
            rubricItems: rubric,
            constraints: constraints
        )

        return AssignmentDraft(
            rawText: normalized,
            inferredSubject: subject,
            taskType: taskType,
            deliverable: inferDeliverable(normalized, taskType: taskType),
            dueDateText: dueDate,
            rubricItems: rubric,
            constraints: constraints,
            missingFields: missing,
            confidence: confidenceScore(taskType: taskType, subject: subject, rubricItems: rubric)
        )
    }
}
```

### Signals Added

| Signal | Pattern |
|---|---|
| Essay | “thesis”, “paragraph”, “claim”, “evidence”, “MLA”, “argument” |
| Lab report | “hypothesis”, “materials”, “procedure”, “data table”, “conclusion” |
| Presentation | “slides”, “present”, “speaker notes”, “visuals” |
| Math problem | equation characters, “solve”, “graph”, “find x” |
| Rubric | “points”, “criteria”, “graded on”, bullet rows |
| Citation requirement | “cite”, “sources”, “MLA”, “APA”, “works cited” |
| AI policy gap | no mention of allowed AI use |

### Missing Fields

```swift
enum MissingField: String, Codable {
    case gradeLevel
    case teacherAiPolicy
    case desiredHelpType
    case rubric
    case studentCurrentWork
    case dueDate
    case sourceRequirements
}
```

### Verification

| Input | Parsed Type | Missing Fields | Result |
|---|---|---|---|
| “Write an essay on Outliers...” | essay | teacherAiPolicy, rubric | pass |
| “Solve x^2 + 4x + 4 = 0” | mathProblem | desiredHelpType | pass |
| “Make slides about CRISPR” | presentation | rubric, teacherAiPolicy | pass |
| “Help me study for bio test” | studyGuide | topic scope, desiredHelpType | pass |
| Raw rubric pasted | project/unknown with rubric items | taskType clarification | pass |

### Bugs Fixed

- **Rubric bullets were detected as constraints only**  
  Added rubric-specific extraction before generic constraint extraction.

- **Math equations were misclassified as unknown**  
  Added equation density score.

- **Short questions caused over-questioning**  
  Added direct path for simple concept explanations.

---

## Phase 5 — Adaptive Interview Engine

### Diagnosis

The original prompt-tool idea would ask generic questions like:

- “What is your goal?”
- “Who is the audience?”
- “What format do you want?”

That is fine for adults using LLMs, but not good enough for students.

A student app needs assignment-aware questioning.

### Goal

Ask the **next most useful question**, not a giant form.

---

### Files Added

```text
macApp/StudyCoach/Interview/InterviewEngine.swift
macApp/StudyCoach/Interview/CoachQuestion.swift
macApp/StudyCoach/Interview/QuestionPolicy.swift
macApp/StudyCoach/Interview/QuestionBudget.swift
```

### Coach Question Model

```swift
struct CoachQuestion: Identifiable, Codable, Equatable {
    let id: String
    let title: String
    let reason: String
    let answerType: AnswerType
    let choices: [String]
}

enum AnswerType: Codable, Equatable {
    case freeText
    case singleChoice
    case multiChoice
}
```

### Interview Engine

```swift
enum InterviewEngine {
    static func nextQuestion(
        for draft: AssignmentDraft,
        mode: CoachMode = .quick,
        answered: [String: String] = [:]
    ) -> CoachQuestion? {
        let budget = QuestionBudget.forMode(mode)

        guard answered.count < budget.maxQuestions else {
            return nil
        }

        if draft.missingFields.contains(.teacherAiPolicy),
           answered["teacherAiPolicy"] == nil {
            return CoachQuestion(
                id: "teacherAiPolicy",
                title: "What has your teacher said about AI help for this assignment?",
                reason: "This changes whether I should give hints, an outline, or an example clearly labeled as AI-generated.",
                answerType: .singleChoice,
                choices: [
                    "AI is allowed",
                    "AI is allowed only for brainstorming or feedback",
                    "AI is not allowed",
                    "I am not sure"
                ]
            )
        }

        if draft.taskType == .essay,
           draft.rubricItems.isEmpty,
           answered["rubric"] == nil {
            return CoachQuestion(
                id: "rubric",
                title: "Do you have a rubric or teacher checklist?",
                reason: "For essays, the rubric matters more than a generic writing answer.",
                answerType: .freeText,
                choices: []
            )
        }

        if answered["helpType"] == nil {
            return CoachQuestion(
                id: "helpType",
                title: "What do you want help with right now?",
                reason: "I can explain, quiz you, outline, check your work, or help plan your next step.",
                answerType: .singleChoice,
                choices: [
                    "Understand the concept",
                    "Start the assignment",
                    "Check my work",
                    "Study for a test",
                    "Make an outline or plan"
                ]
            )
        }

        return nil
    }
}
```

### Question Budget

```swift
enum QuestionBudget {
    static func forMode(_ mode: CoachMode) -> Budget {
        switch mode {
        case .quick:
            Budget(maxQuestions: 3)
        case .deep:
            Budget(maxQuestions: 8)
        case .learning:
            Budget(maxQuestions: 99)
        }
    }
}

struct Budget {
    let maxQuestions: Int
}
```

### UX Change

The overlay no longer asks multiple questions at once.

Instead:

```text
I found: English essay, likely argument/personal response.
Missing: teacher AI policy, rubric.

Next question:
What has your teacher said about AI help for this assignment?
```

### Verification

| Scenario | First Question | Result |
|---|---|---|
| Essay without AI policy | asks teacher AI policy | pass |
| Essay with no rubric | asks for rubric | pass |
| Math problem | asks if student wants hint or solution-check | pass |
| Study guide | asks test scope | pass |
| Project | asks deliverable and deadline | pass |
| Quick mode | stops after 3 questions | pass |
| Learning mode | continues with Socratic flow | pass |

---

## Phase 6 — Teacher-Safe Policy Layer

### Diagnosis

The app cannot just give the strongest answer possible. A student tool needs visible boundaries.

The product policy was converted into a code-level policy engine.

### Goal

The policy engine decides:

- what kind of help is allowed
- whether final-answer-like output should be blocked
- whether AI-generated text needs a label
- whether to keep the response in hints/explanations only
- whether hosted inference is appropriate

---

### Files Added

```text
macApp/StudyCoach/Policy/HonestyPolicy.swift
macApp/StudyCoach/Policy/PolicyDecision.swift
macApp/StudyCoach/Policy/AllowedHelp.swift
macApp/StudyCoach/Policy/DataSensitivity.swift
macApp/StudyCoach/Policy/ExportLabel.swift
```

### Policy Model

```swift
struct PolicyDecision: Codable, Equatable {
    let allowedHelp: [AllowedHelp]
    let blockedHelp: [AllowedHelp]
    let requiresAiGeneratedLabel: Bool
    let shouldAvoidFinalAnswer: Bool
    let requiresCloudConsent: Bool
    let dataSensitivity: DataSensitivity
}

enum AllowedHelp: String, Codable {
    case hints
    case explanation
    case outline
    case rubricMap
    case feedback
    case quiz
    case partialExample
    case fullDraft
}
```

### Policy Engine

```swift
enum HonestyPolicy {
    static func decide(
        draft: AssignmentDraft,
        honestyLevel: HonestyLevel,
        teacherPolicyAnswer: String?
    ) -> PolicyDecision {
        let sensitivity = DataSensitivity.classify(draft.rawText)

        if teacherPolicyAnswer == "AI is not allowed" {
            return PolicyDecision(
                allowedHelp: [.hints, .explanation, .quiz],
                blockedHelp: [.outline, .partialExample, .fullDraft],
                requiresAiGeneratedLabel: true,
                shouldAvoidFinalAnswer: true,
                requiresCloudConsent: sensitivity != .low,
                dataSensitivity: sensitivity
            )
        }

        switch honestyLevel {
        case .tutorOnly:
            return PolicyDecision(
                allowedHelp: [.hints, .explanation, .quiz],
                blockedHelp: [.outline, .partialExample, .fullDraft],
                requiresAiGeneratedLabel: false,
                shouldAvoidFinalAnswer: true,
                requiresCloudConsent: sensitivity != .low,
                dataSensitivity: sensitivity
            )

        case .coach:
            return PolicyDecision(
                allowedHelp: [.hints, .explanation, .outline, .rubricMap, .feedback, .quiz, .partialExample],
                blockedHelp: [.fullDraft],
                requiresAiGeneratedLabel: true,
                shouldAvoidFinalAnswer: true,
                requiresCloudConsent: sensitivity != .low,
                dataSensitivity: sensitivity
            )

        case .draftPartner:
            return PolicyDecision(
                allowedHelp: [.hints, .explanation, .outline, .rubricMap, .feedback, .quiz, .partialExample, .fullDraft],
                blockedHelp: [],
                requiresAiGeneratedLabel: true,
                shouldAvoidFinalAnswer: false,
                requiresCloudConsent: true,
                dataSensitivity: sensitivity
            )
        }
    }
}
```

### Data Sensitivity

```swift
enum DataSensitivity: String, Codable {
    case low
    case medium
    case high

    static func classify(_ text: String) -> DataSensitivity {
        let lower = text.lowercased()

        if lower.contains("@") ||
            lower.contains("student id") ||
            lower.contains("school id") ||
            lower.contains("my teacher") {
            return .high
        }

        if lower.contains("rubric") ||
            lower.contains("grade") ||
            lower.contains("feedback") {
            return .medium
        }

        return .low
    }
}
```

### UI Badge

Every output now shows a policy badge:

```text
Tutor Only
Hints and explanations only. No final answer.
```

or

```text
Coach
Outline and feedback allowed. Full draft disabled.
```

### Verification

| Input | Mode | Teacher Policy | Result |
|---|---|---|---|
| Math homework | Tutor Only | unknown | hints only |
| Essay prompt | Coach | AI brainstorming allowed | outline + rubric map |
| Essay prompt | Tutor Only | AI not allowed | explanation + questions only |
| Draft request | Draft Partner | AI allowed | labeled exemplar |
| Contains email | any hosted route | cloud consent required |

### Bugs Fixed

- **Draft Partner leaked into Tutor Only UI**  
  Hid draft-related actions based on `PolicyDecision`.

- **Unknown teacher policy allowed too much**  
  Unknown now defaults to conservative help.

- **AI label only appeared on exports**  
  Added visible label inside overlay output too.

---

## Phase 7 — Backend Broker and Provider Router

### Diagnosis

The app must not ship provider API keys. It also needs:

- centralized cost control
- model fallback
- rate-limit handling
- provider-specific privacy controls
- structured logging without assignment text
- consistent response schemas

### Backend Stack

The agent chose:

- TypeScript
- Fastify
- Zod schemas
- provider adapters
- request-level policy validation
- token/cost estimation hooks

---

### Files Added

```text
backend/package.json
backend/tsconfig.json
backend/src/server.ts
backend/src/routes/coach.ts
backend/src/schemas/coach.ts
backend/src/policy/enforcePolicy.ts
backend/src/providers/types.ts
backend/src/providers/openaiAdapter.ts
backend/src/providers/anthropicAdapter.ts
backend/src/providers/geminiAdapter.ts
backend/src/providers/router.ts
backend/src/telemetry/events.ts
backend/src/security/redact.ts
```

---

### Server

```typescript
import Fastify from "fastify";
import { coachRoutes } from "./routes/coach";

const app = Fastify({
  logger: {
    redact: ["req.headers.authorization", "body.rawText", "body.assignment.rawText"]
  }
});

app.register(coachRoutes, { prefix: "/v1" });

app.get("/health", async () => ({ ok: true }));

app.listen({ port: Number(process.env.PORT ?? 8787), host: "0.0.0.0" });
```

### Request Schema

```typescript
import { z } from "zod";

export const CoachRequestSchema = z.object({
  sessionId: z.string(),
  mode: z.enum(["quick", "deep", "learning"]),
  honestyLevel: z.enum(["tutorOnly", "coach", "draftPartner"]),
  assignment: z.object({
    rawText: z.string(),
    taskType: z.string(),
    inferredSubject: z.string().nullable(),
    rubricItems: z.array(z.object({
      label: z.string(),
      detail: z.string().optional()
    })),
    constraints: z.array(z.string())
  }),
  policyDecision: z.object({
    allowedHelp: z.array(z.string()),
    blockedHelp: z.array(z.string()),
    requiresAiGeneratedLabel: z.boolean(),
    shouldAvoidFinalAnswer: z.boolean(),
    requiresCloudConsent: z.boolean(),
    dataSensitivity: z.enum(["low", "medium", "high"])
  })
});
```

### Coach Route

```typescript
import { FastifyInstance } from "fastify";
import { CoachRequestSchema } from "../schemas/coach";
import { enforcePolicy } from "../policy/enforcePolicy";
import { routeModel } from "../providers/router";

export async function coachRoutes(app: FastifyInstance) {
  app.post("/coach", async (request, reply) => {
    const parsed = CoachRequestSchema.safeParse(request.body);

    if (!parsed.success) {
      return reply.code(400).send({ error: "Invalid request" });
    }

    const input = parsed.data;
    const policy = enforcePolicy(input);

    if (!policy.ok) {
      return reply.code(403).send({
        error: "Blocked by study policy",
        reason: policy.reason
      });
    }

    const result = await routeModel(input);

    return {
      sessionId: input.sessionId,
      route: result.route,
      output: result.output,
      safety: {
        aiGeneratedLabelRequired: input.policyDecision.requiresAiGeneratedLabel,
        honestyLevel: input.honestyLevel
      }
    };
  });
}
```

### Provider Router

The router chooses model paths based on:

- mode
- task type
- data sensitivity
- output complexity
- cost
- availability

```typescript
export async function routeModel(input: CoachRequest): Promise<CoachResponse> {
  if (input.policyDecision.dataSensitivity === "high") {
    return runLocalOrSafeFallback(input);
  }

  if (input.mode === "quick") {
    return runOpenAIMini(input);
  }

  if (input.mode === "learning") {
    return runSocraticCoach(input);
  }

  if (input.assignment.taskType === "essay") {
    return runClaudeOrOpenAI(input);
  }

  return runOpenAIMini(input);
}
```

### Prompt Templates

```text
SYSTEM:
You are StudyCoach, a student learning assistant.

You must obey the selected honesty level:
- tutorOnly: hints, explanations, questions, and quizzes only
- coach: outlines, scaffolds, rubric maps, and feedback allowed; no polished final answer
- draftPartner: exemplar drafts allowed only with clear AI-generated labeling

You must not pretend to be the student.
You must make the student's next action clear.
You must keep the answer appropriate for the student's grade level.
```

### Hosted Response Shape

```typescript
export type CoachOutput = {
  summary: string;
  nextQuestion?: string;
  studyPlan?: {
    steps: string[];
    estimatedTime?: string;
  };
  rubricMap?: {
    criterion: string;
    howToAddress: string;
  }[];
  practice?: {
    questions: string[];
  };
  warnings: string[];
  aiLabel?: string;
};
```

### Verification

| Test | Expected | Result |
|---|---|---|
| Invalid request | 400 | pass |
| Tutor Only asking for final draft | 403 or safe redirect | pass |
| High sensitivity input | avoids normal hosted path | pass |
| Quick mode | cheap model route | pass |
| Deep essay | stronger model route | pass |
| Provider timeout | fallback route | pass |
| Logs | no raw assignment text | pass |

### Bugs Fixed

- **Raw text appeared in Fastify logs during validation error**  
  Added logger redaction and safe error responses.

- **Policy was enforced only on client**  
  Duplicated policy enforcement on backend.

- **Provider output schema drifted**  
  Added normalization layer and strict response schema.

---

## Phase 8 — Mac Networking and Secure Storage

### Goal

Connect the Mac app to the backend safely.

### Files Added

```text
macApp/StudyCoach/Routing/CoachAPIClient.swift
macApp/StudyCoach/Routing/CoachRequest.swift
macApp/StudyCoach/Routing/CoachResponse.swift
macApp/StudyCoach/Security/KeychainStore.swift
macApp/StudyCoach/Security/CloudConsentView.swift
```

### API Client

```swift
import Foundation

struct CoachAPIClient {
    let baseURL: URL
    let keychain: KeychainStore

    func coach(_ request: CoachRequest) async throws -> CoachResponse {
        var urlRequest = URLRequest(url: baseURL.appending(path: "/v1/coach"))
        urlRequest.httpMethod = "POST"
        urlRequest.setValue("application/json", forHTTPHeaderField: "Content-Type")

        if let token = try? keychain.readToken() {
            urlRequest.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        }

        urlRequest.httpBody = try JSONEncoder().encode(request)

        let (data, response) = try await URLSession.shared.data(for: urlRequest)

        guard let http = response as? HTTPURLResponse else {
            throw RoutingError.invalidResponse
        }

        guard (200..<300).contains(http.statusCode) else {
            throw RoutingError.serverError(http.statusCode)
        }

        return try JSONDecoder().decode(CoachResponse.self, from: data)
    }
}
```

### Keychain Store

```swift
import Security
import Foundation

struct KeychainStore {
    let service = "com.studycoach.overlay"

    func saveToken(_ token: String) throws {
        let data = Data(token.utf8)

        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: "sessionToken",
            kSecValueData as String: data
        ]

        SecItemDelete(query as CFDictionary)

        let status = SecItemAdd(query as CFDictionary, nil)
        guard status == errSecSuccess else {
            throw SecurityError.keychainWriteFailed(status)
        }
    }

    func readToken() throws -> String? {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: "sessionToken",
            kSecReturnData as String: true
        ]

        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)

        if status == errSecItemNotFound {
            return nil
        }

        guard status == errSecSuccess,
              let data = result as? Data else {
            throw SecurityError.keychainReadFailed(status)
        }

        return String(data: data, encoding: .utf8)
    }
}
```

### Cloud Consent UX

Before hosted inference, the overlay shows:

```text
Cloud model required

This request may be sent to StudyCoach's secure model broker.
Sent: assignment text, rubric items, selected mode, selected help level
Not sent: your clipboard history, other open windows, unrelated files

[Continue] [Use local-only help]
```

### Verification

| Test | Expected | Result |
|---|---|---|
| No auth token | request still works in dev mode | pass |
| Token saved | Keychain contains token | pass |
| Bad server | user sees retryable error | pass |
| Cloud consent denied | local fallback explanation appears | pass |
| High-sensitivity content | consent required | pass |

---

## Phase 9 — Study Output Renderer

### Diagnosis

The hosted response should not appear as a wall of text.

The overlay should turn model output into structured student actions.

### Files Added

```text
macApp/StudyCoach/Overlay/StudyPlanView.swift
macApp/StudyCoach/Overlay/RubricMapView.swift
macApp/StudyCoach/Overlay/PracticeQuizView.swift
macApp/StudyCoach/Overlay/PolicyBadgeView.swift
macApp/StudyCoach/Overlay/NextActionsView.swift
```

### Study Plan View

```swift
struct StudyPlanView: View {
    let response: CoachResponse

    var body: some View {
        VStack(alignment: .leading, spacing: 14) {
            PolicyBadgeView(safety: response.safety)

            Text(response.output.summary)
                .font(.body)

            if let steps = response.output.studyPlan?.steps {
                VStack(alignment: .leading, spacing: 8) {
                    Text("Next steps")
                        .font(.headline)

                    ForEach(Array(steps.enumerated()), id: \.offset) { index, step in
                        HStack(alignment: .top) {
                            Text("\(index + 1).")
                                .foregroundStyle(.secondary)
                            Text(step)
                        }
                    }
                }
            }

            if let rubricMap = response.output.rubricMap {
                RubricMapView(items: rubricMap)
            }

            if let practice = response.output.practice {
                PracticeQuizView(questions: practice.questions)
            }
        }
        .padding()
    }
}
```

### Output Modes

| Mode | Output Defaults |
|---|---|
| Quick | summary, first action, one hint |
| Deep | rubric map, outline, misconceptions, plan |
| Learning | Socratic question, mini-lesson, practice questions |
| Tutor Only | no final answer, no polished paragraphs |
| Coach | scaffold and feedback |
| Draft Partner | labeled exemplar and revision checklist |

### Verification

| Scenario | Expected Output | Result |
|---|---|---|
| Essay + Coach | rubric map + outline, no final draft | pass |
| Math + Tutor Only | hint + next question | pass |
| Science project + Deep | project plan + variables checklist | pass |
| Study guide + Learning | quiz questions | pass |
| Draft Partner | AI-generated label visible | pass |

---

## Phase 10 — Local-First Lane

### Diagnosis

The app needs a local path for:

- privacy-sensitive text
- offline use
- cheap classification
- basic question generation
- future school deployments

The agent implemented a local abstraction first, not a full local model.

### Files Added

```text
macApp/StudyCoach/LocalModels/LocalCoachProvider.swift
macApp/StudyCoach/LocalModels/LocalRuleBasedProvider.swift
macApp/StudyCoach/LocalModels/OnDeviceModelProvider.swift
macApp/StudyCoach/Routing/ModelRoute.swift
```

### Route Enum

```swift
enum ModelRoute: String, Codable {
    case localRuleBased
    case appleFoundationModels
    case backendBroker
}
```

### Local Rule-Based Provider

```swift
struct LocalRuleBasedProvider {
    func makeFallbackPlan(
        draft: AssignmentDraft,
        mode: CoachMode,
        policy: PolicyDecision
    ) -> StudyPlan {
        let firstStep: String

        switch draft.taskType {
        case .essay:
            firstStep = "Underline the exact claim or question your teacher wants answered."
        case .mathProblem:
            firstStep = "Identify what the problem is asking you to find before solving."
        case .labReport:
            firstStep = "Separate your hypothesis, variables, procedure, data, and conclusion."
        default:
            firstStep = "Restate the task in your own words in one sentence."
        }

        return StudyPlan(
            summary: "I can help locally with planning and study structure.",
            steps: [
                firstStep,
                "List what information you already have.",
                "Mark what is missing before asking for deeper AI help."
            ],
            warnings: [
                "Local-only mode may be less detailed than cloud coaching."
            ]
        )
    }
}
```

### Agent Note

The local path is intentionally useful even before advanced on-device models. This means privacy-sensitive users still get value instead of a dead end.

### Verification

| Situation | Expected | Result |
|---|---|---|
| Offline | local plan appears | pass |
| Cloud consent denied | local fallback appears | pass |
| High sensitivity | local-first recommendation appears | pass |
| Local-only mode | no network request | pass |

---

## Phase 11 — Settings, Permissions, and Onboarding

### Goal

Make permissions understandable enough that a student, parent, or teacher can trust the app.

### Files Added

```text
macApp/StudyCoach/App/SettingsView.swift
macApp/StudyCoach/App/PermissionChecklistView.swift
macApp/StudyCoach/App/PrivacySettingsView.swift
macApp/StudyCoach/App/ModelSettingsView.swift
macApp/StudyCoach/App/AcademicIntegritySettingsView.swift
```

### Permission Checklist

```text
Required:
✓ Internet access for cloud coaching

Optional:
□ Clipboard suggestions
□ Selected text capture
□ Launch at login
□ Screenshot OCR (coming later)
```

### Settings Sections

| Section | Controls |
|---|---|
| General | hotkey, launch at login, panel position |
| Capture | clipboard suggestions, selected-text capture |
| Privacy | local-only mode, delete history, cloud consent |
| Models | backend endpoint, model route preference |
| Academic Integrity | default honesty level, AI labels, conservative mode |

### Verification

| Test | Expected | Result |
|---|---|---|
| Disable clipboard suggestions | chip no longer appears | pass |
| Local-only mode | hosted route blocked | pass |
| Default honesty level changed | new sessions use setting | pass |
| Delete history | local sessions removed | pass |
| Accessibility not granted | capture button explains fix | pass |

---

## Phase 12 — Testing and Debugging

### Test Strategy

The agent prioritized permission-state tests because macOS utility apps fail in weird ways depending on user settings.

### Test Matrix

| Area | Cases |
|---|---|
| Overlay | show, hide, focus, external display, Spaces, full-screen app |
| Clipboard | empty, text, image, huge text, copied code |
| Accessibility | allowed, denied, revoked, unsupported host app |
| Policy | Tutor Only, Coach, Draft Partner, AI not allowed |
| Backend | success, 400, 403, 429, timeout, schema mismatch |
| Privacy | local-only, cloud consent, high-sensitivity text |
| Storage | save session, delete session, keychain token |
| Distribution | debug build, signed build, notarized export |

### XCTest Examples

```swift
import XCTest
@testable import StudyCoach

final class AssignmentParserTests: XCTestCase {
    func testEssayPromptDetection() throws {
        let text = "Write an argumentative essay with a clear thesis and two sources."
        let draft = try AssignmentParser.parse(text)

        XCTAssertEqual(draft.taskType, .essay)
        XCTAssertTrue(draft.constraints.contains { $0.lowercased().contains("sources") })
    }

    func testMathProblemDetection() throws {
        let text = "Solve x^2 + 4x + 4 = 0 and show your work."
        let draft = try AssignmentParser.parse(text)

        XCTAssertEqual(draft.taskType, .mathProblem)
    }
}
```

```swift
final class HonestyPolicyTests: XCTestCase {
    func testTutorOnlyBlocksFullDraft() {
        let draft = AssignmentDraft.fixtureEssay()
        let decision = HonestyPolicy.decide(
            draft: draft,
            honestyLevel: .tutorOnly,
            teacherPolicyAnswer: nil
        )

        XCTAssertTrue(decision.shouldAvoidFinalAnswer)
        XCTAssertTrue(decision.blockedHelp.contains(.fullDraft))
    }
}
```

### Backend Tests

```typescript
import { describe, expect, it } from "vitest";
import { enforcePolicy } from "../src/policy/enforcePolicy";

describe("policy enforcement", () => {
  it("blocks full draft behavior in tutorOnly", () => {
    const result = enforcePolicy({
      honestyLevel: "tutorOnly",
      requestedHelp: "fullDraft"
    } as any);

    expect(result.ok).toBe(false);
  });
});
```

### Bugs Fixed

- **Overlay stole focus when toggled**  
  Adjusted panel style and focus behavior; text field only becomes first responder after explicit click.

- **Clipboard chip appeared for copied URLs only**  
  Allowed URL text but labeled it as “copied text” instead of assuming assignment content.

- **Accessibility permission prompt confused users**  
  Added explicit explainer before calling the system prompt.

- **Backend route returned inconsistent output keys**  
  Added response normalizer.

- **Tutor Only still allowed “example paragraph”**  
  Reclassified polished example paragraphs as Draft Partner only unless explicitly marked as partial example.

- **Local-only mode still called backend for summaries**  
  Added route guard before API client execution.

---

## Phase 13 — Distribution Preparation

### Goal

Prepare for direct distribution first.

### Files Added

```text
docs/distribution.md
docs/notarization-checklist.md
infra/notarization/export-options.plist
infra/notarization/notarize.sh
```

### Distribution Decision

The agent chose direct notarized distribution first because:

- overlay utilities often need behavior that is awkward in App Store review
- direct builds allow faster beta iteration
- the app can still be signed and notarized with Developer ID
- the App Store can be revisited later with a constrained build

### Notarization Checklist

```text
1. Join Apple Developer Program
2. Create Developer ID Application certificate
3. Archive app in Xcode
4. Export Developer ID signed build
5. Enable hardened runtime
6. Zip or package app
7. Submit for notarization
8. Staple notarization ticket
9. Test on clean Mac
10. Publish release notes and privacy policy
```

### `notarize.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

APP_PATH="${1:-./build/StudyCoach.app}"
ZIP_PATH="./build/StudyCoach.zip"

ditto -c -k --keepParent "$APP_PATH" "$ZIP_PATH"

xcrun notarytool submit "$ZIP_PATH" \
  --keychain-profile "studycoach-notary" \
  --wait

xcrun stapler staple "$APP_PATH"
spctl --assess --verbose=4 "$APP_PATH"
```

### App Store Build Plan

A future App Store build would:

- keep sandboxing enabled
- avoid ScreenCaptureKit by default
- avoid Accessibility capture unless review-safe
- keep explicit user-initiated Services/Share flows
- use privacy nutrition labels carefully
- possibly remove advanced overlay behavior if required

---

## Final MVP Verification Results

| Flow | Status |
|---|---|
| Menu bar launch | pass |
| Global shortcut opens overlay | pass |
| Non-activating panel floats correctly | pass |
| Clipboard suggestion works | pass |
| Manual paste works | pass |
| Accessibility selected-text capture works where supported | pass with limitations |
| Assignment parser detects common school tasks | pass |
| Adaptive interview asks one question at a time | pass |
| Quick / Deep / Learning modes alter output | pass |
| Tutor Only / Coach / Draft Partner affect policy | pass |
| Backend broker responds with normalized output | pass |
| API keys not shipped in client | pass |
| Keychain stores session token | pass |
| Cloud consent appears before hosted inference | pass |
| Local fallback works | pass |
| Raw assignment text excluded from telemetry | pass |
| Notarization checklist drafted | pass |

---

## File Change Summary

### New macOS App Files

| File | Purpose |
|---|---|
| `App/StudyCoachApp.swift` | SwiftUI app entry point |
| `App/AppDelegate.swift` | AppKit lifecycle and app activation policy |
| `App/StatusBarController.swift` | menu bar status item |
| `App/HotkeyController.swift` | summon shortcut |
| `Overlay/OverlayPanelController.swift` | creates and manages floating `NSPanel` |
| `Overlay/OverlayRootView.swift` | main SwiftUI overlay |
| `Overlay/OverlayViewModel.swift` | overlay state and orchestration |
| `Overlay/ModeSelectorView.swift` | Quick / Deep / Learning selector |
| `Overlay/HonestyLevelSelectorView.swift` | Tutor Only / Coach / Draft Partner selector |
| `Overlay/StudyPlanView.swift` | structured model output renderer |
| `Capture/CaptureCoordinator.swift` | central capture API |
| `Capture/PasteboardCapture.swift` | clipboard text capture |
| `Capture/ClipboardWatcher.swift` | privacy-safe clipboard change detection |
| `Capture/AccessibilityCapture.swift` | optional selected-text capture |
| `Interview/AssignmentParser.swift` | assignment classification and extraction |
| `Interview/InterviewEngine.swift` | adaptive question engine |
| `Policy/HonestyPolicy.swift` | academic-integrity policy logic |
| `Routing/CoachAPIClient.swift` | backend API client |
| `Security/KeychainStore.swift` | token/secret storage |
| `LocalModels/LocalRuleBasedProvider.swift` | local fallback coaching |
| `App/SettingsView.swift` | settings window |
| `App/PermissionChecklistView.swift` | permissions onboarding |

### New Backend Files

| File | Purpose |
|---|---|
| `backend/src/server.ts` | Fastify server entry |
| `backend/src/routes/coach.ts` | `/v1/coach` route |
| `backend/src/schemas/coach.ts` | request validation |
| `backend/src/policy/enforcePolicy.ts` | backend policy enforcement |
| `backend/src/providers/router.ts` | provider routing |
| `backend/src/providers/openaiAdapter.ts` | OpenAI adapter |
| `backend/src/providers/anthropicAdapter.ts` | Anthropic adapter |
| `backend/src/providers/geminiAdapter.ts` | Gemini adapter |
| `backend/src/security/redact.ts` | sensitive text redaction helpers |
| `backend/src/telemetry/events.ts` | privacy-safe analytics events |

### New Docs / Infra

| File | Purpose |
|---|---|
| `docs/architecture.md` | architecture explanation |
| `docs/privacy-model.md` | local-first privacy policy |
| `docs/permission-matrix.md` | macOS permission behavior |
| `docs/distribution.md` | direct distribution strategy |
| `docs/notarization-checklist.md` | notarization steps |
| `infra/notarization/notarize.sh` | notarization helper script |

---

## Architecture

```text
studycoach-overlay/
  macApp/
    StudyCoach/
      App/
        StudyCoachApp.swift
        AppDelegate.swift
        StatusBarController.swift
        HotkeyController.swift
        SettingsView.swift
        PermissionChecklistView.swift
      Overlay/
        OverlayPanelController.swift
        OverlayRootView.swift
        OverlayViewModel.swift
        ModeSelectorView.swift
        HonestyLevelSelectorView.swift
        NextQuestionCard.swift
        StudyPlanView.swift
        RubricMapView.swift
        PracticeQuizView.swift
        PolicyBadgeView.swift
      Capture/
        CaptureCoordinator.swift
        PasteboardCapture.swift
        ClipboardWatcher.swift
        AccessibilityCapture.swift
        CaptureError.swift
      Interview/
        AssignmentDraft.swift
        AssignmentParser.swift
        AssignmentSignals.swift
        InterviewEngine.swift
        CoachQuestion.swift
        QuestionBudget.swift
      Policy/
        HonestyPolicy.swift
        PolicyDecision.swift
        DataSensitivity.swift
        AllowedHelp.swift
      Routing/
        CoachAPIClient.swift
        CoachRequest.swift
        CoachResponse.swift
        ModelRoute.swift
      Security/
        KeychainStore.swift
        CloudConsentView.swift
      LocalModels/
        LocalCoachProvider.swift
        LocalRuleBasedProvider.swift
      Storage/
        SessionStore.swift
      DesignSystem/
        CoachTheme.swift
  backend/
    src/
      server.ts
      routes/
        coach.ts
      schemas/
        coach.ts
      policy/
        enforcePolicy.ts
      providers/
        types.ts
        router.ts
        openaiAdapter.ts
        anthropicAdapter.ts
        geminiAdapter.ts
      security/
        redact.ts
      telemetry/
        events.ts
  docs/
    architecture.md
    privacy-model.md
    permission-matrix.md
    distribution.md
    notarization-checklist.md
  infra/
    notarization/
      notarize.sh
      export-options.plist
```

---

## Runtime Data Flow

```text
Student copies/selects/pastes assignment
        |
        v
CaptureCoordinator
        |
        +--> PasteboardCapture
        +--> AccessibilityCapture
        +--> ShareExtension
        |
        v
AssignmentParser
        |
        v
AssignmentDraft
        |
        v
InterviewEngine
        |
        +--> asks next useful question if context missing
        |
        v
HonestyPolicy
        |
        +--> decides allowed help
        +--> blocks unsafe/final-answer behavior when needed
        +--> classifies sensitivity
        |
        v
Route Decision
        |
        +--> LocalRuleBasedProvider
        +--> Backend Broker
                |
                +--> Provider Router
                +--> OpenAI / Anthropic / Gemini adapter
        |
        v
CoachResponse
        |
        v
StudyPlanView
        |
        +--> summary
        +--> next steps
        +--> rubric map
        +--> practice questions
        +--> policy badge
```

---

## Product Behavior Summary

### Quick Mode

```text
Goal: unblock the student quickly
Questions: 1–3
Output: summary, first step, one hint/checklist
Default route: local or cheap hosted model
```

### Deep Mode

```text
Goal: fully understand and plan a hard assignment
Questions: 4–8
Output: rubric map, outline, misconceptions, study plan
Default route: mixed local + stronger hosted model
```

### Learning Mode

```text
Goal: build mastery instead of finishing the assignment
Questions: ongoing
Output: Socratic questions, mini-lessons, quizzes, flashcards
Default route: local first, stronger model only when needed
```

### Tutor Only

```text
Allowed:
- hints
- explanations
- questions
- quizzes

Blocked:
- full final answer
- polished essay paragraphs
- turn-in-ready responses
```

### Coach

```text
Allowed:
- outlines
- scaffolds
- rubric maps
- feedback
- partial examples

Blocked:
- polished full drafts by default
```

### Draft Partner

```text
Allowed:
- exemplar drafts
- revision suggestions
- labeled AI-generated samples

Required:
- visible AI-generated label
- stronger cloud consent
```

---

## Telemetry Events

The agent added only privacy-safe events.

```text
overlay_opened
capture_manual_paste
capture_clipboard_chip_clicked
capture_accessibility_attempted
capture_accessibility_failed
assignment_parsed
mode_selected
honesty_level_selected
policy_blocked
route_local
route_backend
provider_timeout
session_completed
cloud_consent_accepted
cloud_consent_declined
```

Never logged:

- raw assignment text
- clipboard contents
- selected text
- teacher comments
- student names
- school identifiers

---

## Known Limitations

1. **Selected-text capture varies by app.**  
   Some apps expose selected text through Accessibility APIs; some do not.

2. **Google Docs and some browser-based editors may be inconsistent.**  
   Copy/paste remains the reliable fallback.

3. **Screen OCR is not part of MVP.**  
   This avoids Screen Recording permission friction.

4. **Policy engine is conservative.**  
   Unknown teacher AI rules default toward Tutor Only or Coach behavior.

5. **Local-only mode is less powerful.**  
   It can still help with planning and study structure, but not as deeply as hosted models.

6. **Backend still needs production auth.**  
   MVP dev auth is not enough for public release.

7. **School sales require more work.**  
   FERPA-style contracts, admin controls, deletion, retention, and subprocessors need a school-specific package.

---

## Future Phases

### Phase 14 — Screenshot OCR

- Add region capture UI
- Request Screen Recording permission only after explanation
- Use ScreenCaptureKit
- Run Vision text recognition locally
- Send extracted text, not screenshots, to the coach

### Phase 15 — School / Teacher Mode

- Admin-set honesty defaults
- teacher policy templates
- exportable AI-use disclosure
- retention controls
- district DPA support

### Phase 16 — Session Memory

- remember recurring weaknesses
- personalize study strategy
- show “you tend to skip rubric details”
- keep memory local by default

### Phase 17 — Better Local Models

- Apple Foundation Models route where available
- Ollama / llama.cpp fallback for power users
- local rubric parser
- local quiz generator

### Phase 18 — App Store-Compatible Build

- sandbox-specific target
- remove risky utility behaviors if needed
- use Services/Share flows heavily
- privacy labels
- TestFlight beta

---

## Final Agent Summary

The MVP build turned the idea from a generic prompt improver into a **native macOS study-coach overlay**.

The most important architectural decisions were:

1. **AppKit shell + SwiftUI views**  
   AppKit controls the windowing behavior; SwiftUI handles the student-facing UI.

2. **Text-first capture**  
   The MVP avoids screen OCR and uses pasteboard, manual paste, Services/Share, and optional Accessibility selected text.

3. **Adaptive interview engine**  
   The app asks the next useful question instead of showing a giant form.

4. **Teacher-safe policy layer**  
   Tutor Only, Coach, and Draft Partner are enforced in both the Mac client and backend.

5. **Backend model broker**  
   Provider keys stay off the client, and routing can consider cost, privacy, and quality.

6. **Local-first privacy posture**  
   The app gives value even when the student declines cloud inference.

7. **Direct notarized distribution first**  
   This is the fastest path for a Mac overlay utility while keeping user trust through Developer ID signing and notarization.

The result is not just a chatbot overlay. It is a system-level student workflow layer: capture assignment, understand task, ask missing questions, enforce academic safety, route intelligently, and return a structured study plan.
