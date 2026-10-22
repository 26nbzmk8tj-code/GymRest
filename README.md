# GymRest
A native watchOS workout tracker built
import SwiftUI
import WatchKit
import Combine
import HealthKit

// MARK: - Models

struct Exercise: Identifiable, Hashable {
    let id = UUID()
    let name: String

    static let all: [Exercise] = [
        Exercise(name: "BENCH PRESS"),
        Exercise(name: "SQUAT"),
        Exercise(name: "DEADLIFT"),
        Exercise(name: "LAT PULLDOWN"),
        Exercise(name: "SHOULDER PRESS"),
        Exercise(name: "BICEPS CURL"),
        Exercise(name: "TRICEPS PUSHDOWN")
    ]
}

struct SetRecord: Identifiable, Codable {
    let id: UUID
    let setNumber: Int
    let weight: Double
    let reps: Int

    init(
        id: UUID = UUID(),
        setNumber: Int,
        weight: Double,
        reps: Int
    ) {
        self.id = id
        self.setNumber = setNumber
        self.weight = weight
        self.reps = reps
    }
}

struct WorkoutRecord: Identifiable, Codable {
    let id: UUID
    let date: Date
    let exerciseName: String
    let sets: [SetRecord]

    init(
        id: UUID = UUID(),
        date: Date = Date(),
        exerciseName: String,
        sets: [SetRecord]
    ) {
        self.id = id
        self.date = date
        self.exerciseName = exerciseName
        self.sets = sets
    }
}

// MARK: - Workout Manager

final class WorkoutManager: NSObject, ObservableObject {

    @Published var workoutHistory: [WorkoutRecord] = []
    @Published var heartRate: Double = 0
    @Published var healthWorkoutRunning = false

    private let healthStore = HKHealthStore()

    private var workoutSession: HKWorkoutSession?
    private var workoutBuilder: HKLiveWorkoutBuilder?

    private let heartRateType = HKObjectType.quantityType(
        forIdentifier: .heartRate
    )!

    override init() {
        super.init()
        loadHistory()
    }

    // MARK: HealthKit Authorization

    func requestAuthorization() {

        guard HKHealthStore.isHealthDataAvailable() else {
            return
        }

        let workoutType = HKObjectType.workoutType()

        let readTypes: Set<HKObjectType> = [
            heartRateType,
            HKObjectType.quantityType(
                forIdentifier: .activeEnergyBurned
            )!
        ]

        let shareTypes: Set<HKSampleType> = [
            workoutType
        ]

        healthStore.requestAuthorization(
            toShare: shareTypes,
            read: readTypes
        ) { success, error in

            if let error = error {
                print(
                    "HealthKit authorization error: \(error)"
                )
            }

            print(
                "HealthKit authorization: \(success)"
            )
        }
    }

    // MARK: Start Workout

    func startWorkout() {

        guard HKHealthStore.isHealthDataAvailable() else {
            return
        }

        guard workoutSession == nil else {
            return
        }

        let configuration = HKWorkoutConfiguration()

        configuration.activityType =
            .traditionalStrengthTraining

        configuration.locationType = .indoor

        do {

            let session = try HKWorkoutSession(
                healthStore: healthStore,
                configuration: configuration
            )

            let builder =
                session.associatedWorkoutBuilder()

            session.delegate = self
            builder.delegate = self

            builder.dataSource =
                HKLiveWorkoutDataSource(
                    healthStore: healthStore,
                    workoutConfiguration: configuration
                )

            workoutSession = session
            workoutBuilder = builder

            let startDate = Date()

            session.startActivity(
                with: startDate
            )

            builder.beginCollection(
                withStart: startDate
            ) { success, error in

                if let error = error {
                    print(
                        "Workout collection error: \(error)"
                    )
                }

                print(
                    "Workout collection started: \(success)"
                )
            }

        } catch {

            print(
                "Could not start workout: \(error)"
            )
        }
    }

    // MARK: End Workout

    func endWorkout() {

        guard let session = workoutSession else {
            return
        }

        session.end()
    }

    // MARK: Save History

    func saveWorkout(
        exerciseName: String,
        sets: [SetRecord]
    ) {

        guard !sets.isEmpty else {
            return
        }

        let record = WorkoutRecord(
            exerciseName: exerciseName,
            sets: sets
        )

        workoutHistory.insert(
            record,
            at: 0
        )

        saveHistory()
    }

    private func saveHistory() {

        do {

            let data =
                try JSONEncoder().encode(
                    workoutHistory
                )

            UserDefaults.standard.set(
                data,
                forKey: "GymRestWorkoutHistory"
            )

        } catch {

            print(
                "Could not save history: \(error)"
            )
        }
    }

    private func loadHistory() {

        guard let data =
            UserDefaults.standard.data(
                forKey: "GymRestWorkoutHistory"
            )
        else {
            return
        }

        do {

            workoutHistory =
                try JSONDecoder().decode(
                    [WorkoutRecord].self,
                    from: data
                )

        } catch {

            print(
                "Could not load history: \(error)"
            )
        }
    }

    // MARK: Heart Rate

    private func updateHeartRate() {

        guard let builder = workoutBuilder else {
            return
        }

        guard let statistics =
            builder.statistics(
                for: heartRateType
            )
        else {
            return
        }

        guard let quantity =
            statistics.mostRecentQuantity()
        else {
            return
        }

        let unit =
            HKUnit.count().unitDivided(
                by: HKUnit.minute()
            )

        let bpm =
            quantity.doubleValue(
                for: unit
            )

        DispatchQueue.main.async {

            self.heartRate = bpm
        }
    }
}

// MARK: - HKWorkoutSessionDelegate

extension WorkoutManager:
    HKWorkoutSessionDelegate {

    func workoutSession(
        _ workoutSession: HKWorkoutSession,
        didChangeTo toState: HKWorkoutSessionState,
        from fromState: HKWorkoutSessionState,
        date: Date
    ) {

        DispatchQueue.main.async {

            switch toState {

            case .running:
                self.healthWorkoutRunning = true

            case .ended:
                self.healthWorkoutRunning = false

            case .stopped:
                self.healthWorkoutRunning = false

            case .paused:
                break

            case .prepared:
                break

            case .notStarted:
                break

            @unknown default:
                break
            }
        }

        if toState == .ended {

            guard let builder =
                self.workoutBuilder
            else {
                return
            }

            builder.endCollection(
                withEnd: date
            ) { success, error in

                if let error = error {

                    print(
                        "End collection error: \(error)"
                    )
                }

                print(
                    "Workout collection ended: \(success)"
                )

                builder.finishWorkout {
                    workout, error in

                    if let error = error {

                        print(
                            "Finish workout error: \(error)"
                        )

                    } else {

                        print(
                            "HealthKit workout saved"
                        )
                    }

                    DispatchQueue.main.async {

                        self.workoutSession = nil
                        self.workoutBuilder = nil
                        self.healthWorkoutRunning = false
                        self.heartRate = 0
                    }
                }
            }
        }
    }

    func workoutSession(
        _ workoutSession: HKWorkoutSession,
        didFailWithError error: Error
    ) {

        print(
            "Workout session failed: \(error)"
        )

        DispatchQueue.main.async {

            self.healthWorkoutRunning = false
            self.workoutSession = nil
            self.workoutBuilder = nil
        }
    }
}

// MARK: - HKLiveWorkoutBuilderDelegate

extension WorkoutManager:
    HKLiveWorkoutBuilderDelegate {

    func workoutBuilderDidCollectEvent(
        _ workoutBuilder: HKLiveWorkoutBuilder
    ) {
    }

    func workoutBuilder(
        _ workoutBuilder: HKLiveWorkoutBuilder,
        didCollectDataOf collectedTypes:
            Set<HKSampleType>
    ) {

        if collectedTypes.contains(
            heartRateType
        ) {

            updateHeartRate()
        }
    }
}

// MARK: - Content View

struct ContentView: View {

    @StateObject private var manager =
        WorkoutManager()

    @State private var showingHistory =
        false

    var body: some View {

        NavigationStack {

            VStack(spacing: 10) {

                Text("GYMREST")
                    .font(.headline)
                    .fontWeight(.bold)

                Button {

                    showingHistory = true

                } label: {

                    Label(
                        "HISTORY",
                        systemImage:
                            "clock.arrow.circlepath"
                    )
                }

                NavigationLink(
                    destination:
                        WorkoutView(
                            manager: manager
                        )
                ) {

                    Text("START WORKOUT")
                        .fontWeight(.bold)
                }
            }
            .padding()
            .sheet(
                isPresented:
                    $showingHistory
            ) {

                HistoryView(
                    manager: manager
                )
            }
        }
    }
}

// MARK: - Workout View

struct WorkoutView: View {

    @ObservedObject var manager:
        WorkoutManager

    @State private var selectedExercise =
        Exercise.all[0]

    @State private var currentSet = 1

    @State private var totalSets = 4

    @State private var weight:
        Double = 60

    @State private var reps:
        Int = 10

    @State private var completedSets:
        [SetRecord] = []

    @State private var restDuration =
        90

    @State private var timeRemaining =
        90

    @State private var isResting =
        false

    @State private var isPaused =
        false

    @State private var targetEndDate:
        Date?

    @State private var showExercises =
        false

    @State private var showSummary =
        false

    // MARK: Previous Workout

    var lastWorkout:
        WorkoutRecord? {

        manager.workoutHistory.first {

            $0.exerciseName ==
                selectedExercise.name
        }
    }

    var previousSet:
        SetRecord? {

        guard let workout =
            lastWorkout
        else {
            return nil
        }

        return workout.sets.first {

            $0.setNumber ==
                currentSet
        }
    }

    // MARK: Body

    var body: some View {

        ScrollView {

            VStack(spacing: 8) {

                exerciseHeader

                heartRateView

                Divider()

                setInfo

                controls

                restSection

                doneButton

                historyButton
            }
            .padding(.horizontal, 8)
            .padding(.vertical, 5)
        }

        .onAppear {

            manager.requestAuthorization()

            if !manager.healthWorkoutRunning {

                manager.startWorkout()
            }

            loadPreviousSet()
        }

        .sheet(
            isPresented:
                $showExercises
        ) {

            ExercisePickerView(
                selectedExercise:
                    $selectedExercise
            )
        }

        .sheet(
            isPresented:
                $showSummary
        ) {

            WorkoutSummaryView(
                exercise:
                    selectedExercise,
                sets:
                    completedSets
            )
        }
    }

    // MARK: Header

    private var exerciseHeader:
        some View {

        VStack(spacing: 2) {

            Text(
                selectedExercise.name
            )
            .font(.headline)
            .multilineTextAlignment(
                .center
            )

            Button(
                "CHANGE EXERCISE"
            ) {

                if manager.healthWorkoutRunning {

                    manager.endWorkout()
                }

                showExercises = true
            }
            .font(.caption2)
        }
    }

    // MARK: Heart Rate

    private var heartRateView:
        some View {

        HStack(spacing: 5) {

            Image(
                systemName:
                    "heart.fill"
            )
            .foregroundColor(.red)

            if manager.heartRate > 0 {

                Text(
                    "\(Int(manager.heartRate)) BPM"
                )
                .font(.caption)
                .fontWeight(
                    .semibold
                )

            } else {

                Text("-- BPM")
                    .font(.caption)
                    .foregroundColor(
                        .secondary
                    )
            }
        }
    }

    // MARK: Set Info

    private var setInfo:
        some View {

        VStack(spacing: 4) {

            Text(
                "SET \(currentSet) / \(totalSets)"
            )
            .font(.headline)

            if let previous =
                previousSet {

                Text(
                    "LAST: \(formatWeight(previous.weight)) × \(previous.reps)"
                )
                .font(.caption2)
                .foregroundColor(
                    .secondary
                )
            }

            HStack(spacing: 8) {

                VStack(spacing: 2) {

                    Text("WEIGHT")
                        .font(.caption2)

                    HStack {

                        Button("-") {

                            weight =
                                max(
                                    0,
                                    weight - 2.5
                                )
                        }

                        Text(
                            formatWeight(
                                weight
                            )
                        )
                        .font(.headline)
                        .frame(
                            minWidth: 45
                        )

                        Button("+") {

                            weight += 2.5
                        }
                    }
                }

                VStack(spacing: 2) {

                    Text("REPS")
                        .font(.caption2)

                    HStack {

                        Button("-") {

                            reps =
                                max(
                                    1,
                                    reps - 1
                                )
                        }

                        Text(
                            "\(reps)"
                        )
                        .font(.headline)
                        .frame(
                            minWidth: 25
                        )

                        Button("+") {

                            reps += 1
                        }
                    }
                }
            }
        }
    }

    // MARK: Controls

    private var controls:
        some View {

        HStack {

            Button("30") {

                restDuration = 30
            }

            Button("60") {

                restDuration = 60
            }

            Button("90") {

                restDuration = 90
            }

            Button("120") {

                restDuration = 120
            }
        }
        .font(.caption2)
    }

    // MARK: Rest Section

    private var restSection:
        some View {

        VStack(spacing: 5) {

            if isResting {

                TimelineView(
                    .periodic(
                        from: .now,
                        by: 1
                    )
                ) { context in

                    let remaining =
                        calculateRemaining(
                            at: context.date
                        )

                    ZStack {

                        Circle()
                            .stroke(
                                lineWidth: 5
                            )
                            .opacity(
                                0.2
                            )

                        Circle()
                            .trim(
                                from: 0,
                                to:
                                    CGFloat(
                                        remaining
                                    ) /
                                    CGFloat(
                                        max(
                                            restDuration,
                                            1
                                        )
                                    )
                            )
                            .stroke(
                                lineWidth: 5
                            )
                            .rotationEffect(
                                .degrees(
                                    -90
                                )
                            )

                        Text(
                            formatTime(
                                remaining
                            )
                        )
                        .font(.headline)
                    }
                    .frame(
                        width: 80,
                        height: 80
                    )
                }

                HStack {

                    Button(
                        isPaused
                        ? "START"
                        : "PAUSE"
                    ) {

                        if isPaused {

                            resumeRest()

                        } else {

                            pauseRest()
                        }
                    }

                    Button("SKIP") {

                        skipRest()
                    }
                }
                .font(.caption)
            }
        }
    }

    // MARK: Calculate Remaining

    private func calculateRemaining(
        at date: Date
    ) -> Int {

        guard isResting else {

            return timeRemaining
        }

        if isPaused {

            return timeRemaining
        }

        guard let endDate =
            targetEndDate
        else {

            return timeRemaining
        }

        let seconds =
            endDate.timeIntervalSince(
                date
            )

        return max(
            0,
            Int(
                ceil(seconds)
            )
        )
    }

    // MARK: Done Button

    private var doneButton:
        some View {

        Button {

            completeSet()

        } label: {

            Text("DONE SET")
                .fontWeight(.bold)
        }
        .disabled(
            isResting
        )
    }

    // MARK: History

    private var historyButton:
        some View {

        NavigationLink(
            destination:
                HistoryView(
                    manager: manager
                )
        ) {

            Text("HISTORY")
                .font(.caption2)
        }
    }

    // MARK: Complete Set

    private func completeSet() {

        let record =
            SetRecord(
                setNumber:
                    currentSet,
                weight:
                    weight,
                reps:
                    reps
            )

        let updatedSets =
            completedSets + [record]

        completedSets =
            updatedSets

        if currentSet >= totalSets {

            manager.saveWorkout(
                exerciseName:
                    selectedExercise.name,
                sets:
                    updatedSets
            )

            manager.endWorkout()

            showSummary = true

        } else {

            startRest()
        }
    }

    // MARK: Start Rest

    private func startRest() {

        timeRemaining =
            restDuration

        targetEndDate =
            Date().addingTimeInterval(
                TimeInterval(
                    restDuration
                )
            )

        isResting = true

        isPaused = false
    }

    // MARK: Pause

    private func pauseRest() {

        if let endDate =
            targetEndDate {

            let seconds =
                endDate.timeIntervalSinceNow

            timeRemaining =
                max(
                    0,
                    Int(
                        ceil(seconds)
                    )
                )
        }

        targetEndDate = nil

        isPaused = true
    }

    // MARK: Resume

    private func resumeRest() {

        targetEndDate =
            Date().addingTimeInterval(
                TimeInterval(
                    timeRemaining
                )
            )

        isPaused = false
    }

    // MARK: Skip

    private func skipRest() {

        timeRemaining = 0

        isResting = false

        isPaused = false

        targetEndDate = nil

        WKInterfaceDevice.current().play(
            .click
        )

        moveToNextSet()
    }

    // MARK: Next Set

    private func moveToNextSet() {

        if currentSet < totalSets {

            currentSet += 1

            loadPreviousSet()
        }
    }

    // MARK: Previous Performance

    private func loadPreviousSet() {

        guard let previous =
            previousSet
        else {
            return
        }

        weight =
            previous.weight

        reps =
            previous.reps
    }

    // MARK: New Workout

    private func newWorkout() {

        currentSet = 1

        completedSets = []

        weight = 60

        reps = 10

        isResting = false

        isPaused = false

        targetEndDate = nil

        timeRemaining =
            restDuration

        if !manager.healthWorkoutRunning {

            manager.startWorkout()
        }

        loadPreviousSet()
    }

    // MARK: Format Weight

    private func formatWeight(
        _ value: Double
    ) -> String {

        if value.truncatingRemainder(
            dividingBy: 1
        ) == 0 {

            return String(
                Int(value)
            )
        }

        return String(
            format: "%.1f",
            value
        )
    }

    // MARK: Format Time

    private func formatTime(
        _ seconds: Int
    ) -> String {

        let minutes =
            seconds / 60

        let secondsPart =
            seconds % 60

        return String(
            format:
                "%02d:%02d",
            minutes,
            secondsPart
        )
    }
}

// MARK: - Exercise Picker

struct ExercisePickerView:
    View {

    @Binding var selectedExercise:
        Exercise

    @Environment(\.dismiss)
    private var dismiss

    var body: some View {

        NavigationStack {

            List {

                ForEach(
                    Exercise.all
                ) { exercise in

                    Button {

                        selectedExercise =
                            exercise

                        dismiss()

                    } label: {

                        HStack {

                            Text(
                                exercise.name
                            )

                            Spacer()

                            if exercise ==
                                selectedExercise {

                                Image(
                                    systemName:
                                        "checkmark"
                                )
                            }
                        }
                    }
                }
            }
            .navigationTitle(
                "EXERCISES"
            )
        }
    }
}

// MARK: - History View

struct HistoryView:
    View {

    @ObservedObject var manager:
        WorkoutManager

    var body: some View {

        List {

            if manager.workoutHistory.isEmpty {

                Text(
                    "NO WORKOUTS YET"
                )
                .foregroundColor(
                    .secondary
                )

            } else {

                ForEach(
                    manager.workoutHistory
                ) { workout in

                    VStack(
                        alignment:
                            .leading,
                        spacing: 4
                    ) {

                        Text(
                            workout.exerciseName
                        )
                        .font(
                            .headline
                        )

                        Text(
                            workout.date.formatted(
                                date:
                                    .abbreviated,
                                time:
                                    .shortened
                            )
                        )
                        .font(
                            .caption2
                        )
                        .foregroundColor(
                            .secondary
                        )

                        ForEach(
                            workout.sets
                        ) { set in

                            Text(
                                "SET \(set.setNumber): \(formatWeight(set.weight)) × \(set.reps)"
                            )
                            .font(
                                .caption
                            )
                        }
                    }
                    .padding(
                        .vertical,
                        3
                    )
                }
            }
        }
        .navigationTitle(
            "HISTORY"
        )
    }

    private func formatWeight(
        _ value: Double
    ) -> String {

        if value.truncatingRemainder(
            dividingBy: 1
        ) == 0 {

            return String(
                Int(value)
            )
        }

        return String(
            format: "%.1f",
            value
        )
    }
}

// MARK: - Summary

struct WorkoutSummaryView:
    View {

    let exercise:
        Exercise

    let sets:
        [SetRecord]

    @Environment(\.dismiss)
    private var dismiss

    var body: some View {

        ScrollView {

            VStack(spacing: 8) {

                Text(
                    "WORKOUT COMPLETE"
                )
                .font(
                    .headline
                )
                .fontWeight(
                    .bold
                )

                Text(
                    exercise.name
                )
                .font(
                    .caption
                )
                .foregroundColor(
                    .secondary
                )

                Divider()

                ForEach(
                    sets
                ) { set in

                    HStack {

                        Text(
                            "SET \(set.setNumber)"
                        )

                        Spacer()

                        Text(
                            "\(formatWeight(set.weight)) × \(set.reps)"
                        )
                        .fontWeight(
                            .semibold
                        )
                    }
                }

                Button("DONE") {

                    dismiss()
                }
                .padding(
                    .top,
                    8
                )
            }
            .padding()
        }
    }

    private func formatWeight(
        _ value: Double
    ) -> String {

        if value.truncatingRemainder(
            dividingBy: 1
        ) == 0 {

            return String(
                Int(value)
            )
        }

        return String(
            format: "%.1f",
            value
        )
    }
}

// MARK: - Preview

struct ContentView_Previews:
    PreviewProvider {

    static var previews:
        some View {

        ContentView()
    }
}
