# CampusLink — Android campus management

CampusLink is a native Android application written in Kotlin. It includes student and administrator sign-in, role-aware campus screens, Firebase-backed records, offline Room caching, and queued uploads/writes.

## Included

- Jetpack Compose + Material 3 UI, with system, light, and dark themes.
- Firebase Authentication with role verification: admins must have the signed Firebase custom claim `admin: true` and an `admin` profile; students need an active `student` profile.
- Firestore repositories for student profiles, courses, attendance, timetable, assignments/submissions, announcements, fees, results, and notifications.
- Room cache scoped to the signed-in account, plus WorkManager to retry queued writes and assignment/profile-image uploads after connectivity returns.
- Firebase Storage for private profile pictures and assignment files.
- Cloud Functions for administrator-created student accounts, account deactivation, targeted notifications, and automatic notifications when assignments, announcements, fees, results, or timetable entries change.
- Firestore and Storage rules that restrict students to their own records or records targeted to their enrolled courses.
- Local Firebase Emulator Suite setup and synthetic sample data.

The Android project is in this directory. It is not an Expo app and does not have a Replit mobile preview.

## Open and build

1. Open `student-campus-android/` in Android Studio.
2. Use JDK 17 and install Android SDK 36 when Android Studio prompts.
3. Choose a Firebase setup below.
4. Sync Gradle and run the `app` configuration on an Android emulator or device.

The development container used to create this source does not include Java or the Android SDK, so the Android APK was not compiled here. Build and run it from Android Studio after setup.

## Firebase project setup

Create a Firebase project, then:

1. Add an Android app with package name `edu.campuslink.app`.
2. Enable **Authentication → Email/Password**.
3. Create Firestore and Storage in the Firebase console.
4. Download the app's `google-services.json` to `app/google-services.json`. Do not commit this file.
5. Deploy rules, indexes, and functions from this directory:

   ```sh
   firebase deploy --only firestore:rules,firestore:indexes,storage
   pnpm --dir firebase/functions install --ignore-workspace
   firebase deploy --only functions
   ```

6. Create the first administrator in Firebase Authentication, then grant the custom claim and initialize their profile. Use an administrator/service-account identity that has permission to set Firebase Auth custom claims. Keep the service account file outside the repository:

   ```sh
   export GOOGLE_APPLICATION_CREDENTIALS=/secure/path/campus-admin-service-account.json
   export GCLOUD_PROJECT=your-firebase-project-id
   export CAMPUS_ADMIN_EMAIL=admin@your-campus.example
   node firebase/functions/scripts/set-admin-claim.mjs
   ```

7. Sign out and back in after assigning the claim. Create later student accounts from **Student management** in the app.

The admin-claim script only prints the target email and never reads or prints credential contents. Never put a service-account key in the Android app or commit it.

### Firestore collections and fields

| Collection | Main fields | Student access |
| --- | --- | --- |
| `users/{uid}` | `role`, `active`, `displayName`, `email`, `studentNumber`, `department`, `yearOfStudy`, `phone`, `photoPath` | Own profile only; can edit a small allowlist of profile fields |
| `courses/{courseId}` | `title`, `courseCode`, `instructor`, `semester`, `credits`, `studentUids` | Enrolled students only |
| `attendance/{id}` | `studentUid`, `courseId`, `courseName`, `date`, `status` | Own records only |
| `timetable/{id}` | `courseId`, `courseName`, `day`, `startTime`, `endTime`, `room`, `visibleToUids` | Targeted students only |
| `assignments/{id}` | `courseId`, `title`, `instructions`, `dueAt` (epoch milliseconds), `visibleToUids` | Targeted students only |
| `submissions/{id}` | `assignmentId`, `studentUid`, `submittedAt`, `status`, `late`, `filePath` | Student's own submissions; admins can review all |
| `announcements/{id}` | `title`, `body`, `audience` (`all` or `selected`), `visibleToUids`, `createdAt` | Campus-wide or targeted |
| `fees/{id}` | `studentUid`, `title`, `amount`, `paidAmount`, `balance`, `status`, `dueDate` | Own records only |
| `payments/{id}` | `studentUid`, `feeId`, `amount`, `method`, `reference`, `recordedAt` | Own payment history only |
| `results/{id}` | `studentUid`, `courseId`, `marks`, `maxMarks`, `grade`, `published` | Own records when published |
| `users/{uid}/notifications/{id}` | `title`, `body`, `type`, `recordId`, `createdAt`, `read` | Own inbox only |
| `users/{uid}/devices/{tokenId}` | `token`, `updatedAt` | Own FCM device tokens |

For course-scoped records, administrators should set the student's UID in `studentUids` or `visibleToUids`. This makes the student query and security rule agree; records without those fields are not visible to students.

The deployed rules rely on Firebase Auth custom claims for admin access. Do not replace this with a client-editable role field. Read `firebase/firestore.rules` and `firebase/storage.rules` before deploying; they are the security boundary for private student data.

## Local sample data with Firebase emulators

The seed script is guarded: it refuses to run without both Auth and Firestore emulator hosts and a `demo-*` project ID. It creates only synthetic identities, with the same local-only demo password `CampusDemo2026!`.

1. Install Firebase CLI and the Functions package in your development environment.
2. From `student-campus-android/`, start the emulators:

   ```sh
   firebase emulators:start --project demo-campuslink-local
   ```

3. In another terminal, seed the emulators:

   ```sh
   FIREBASE_AUTH_EMULATOR_HOST=127.0.0.1:9099 \
   FIRESTORE_EMULATOR_HOST=127.0.0.1:8080 \
   GCLOUD_PROJECT=demo-campuslink-local \
   node firebase/functions/scripts/seed-emulator.mjs
   ```

4. Set `campusFirebaseEmulator=true` in `gradle.properties`, then run the debug app on an Android emulator. The Android emulator reaches host emulators through `10.0.2.2`; for a physical device, set `campusFirebaseEmulatorHost` to the development computer's LAN address.
5. Use `admin@demo.campuslink.test` for admin login or `amina@demo.campuslink.test` for student login. The local-only demo password is the same for both accounts.

Firebase Cloud Messaging does not have an Emulator Suite emulator. The app's Firestore inbox and other emulator-backed functions work locally; real push delivery needs a real Firebase project and FCM configuration.

## Offline behavior and limitations

- Firestore's SDK cache supports previously fetched cloud records. Room provides an additional per-account cache for records used by the screens.
- Changes are queued locally and retried by WorkManager when the device has connectivity. A write can remain pending until the signed-in account regains permission and reaches the server.
- Data loads 100 records per page, with “Load more” for large lists. Search and filters run on downloaded pages; announcements load the first 100 campus-wide and 100 targeted records.
- The app expects a campus administrator to create student accounts; public self-registration is intentionally not exposed.
- Student record updates, grading, payments, and notifications are protected by Firestore rules or callable functions. Real student data should be tested with separate student accounts and the Firebase Emulator Suite before deployment.
