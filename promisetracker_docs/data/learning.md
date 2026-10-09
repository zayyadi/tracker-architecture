# Data Model — Learning (LMS)

Accurate as of: 2026-09-10

## Entity relationship diagram

```mermaid
erDiagram
    COHORTS ||--o{ USERS : enrols
    COHORTS ||--o{ STUDENTS : "pre-enrols"
    MODULES |o--o{ COURSES : groups
    INSTRUCTORS ||--o{ COURSES : teaches
    COURSES ||--o{ QUESTIONS : asks
    QUESTIONS ||--o{ CHOICES : offers
    QUESTIONS ||--o{ ANSWERS : "correct answer"
    COURSES ||--o{ RESOURCES : attaches
    USERS ||--o{ USER_ANSWERS : submits
    USERS ||--o{ LEADER_BOARDS : scores
    USERS ||--o{ EXTRA_SCORES : earns
    USERS ||--o{ COURSE_PROGRESS : watches

    MODULES {
        uuid id PK
        string name
    }
    COURSES {
        uuid id PK
        uuid module_id FK "nullable, conventional"
        uuid instructor_id FK "conventional"
        string title
        string category
        string video_url
        string points
        string audio_type
        enum status "active|archived"
        string transcript_url "nullable"
        string duration "nullable"
    }
    QUESTIONS {
        uuid id PK
        uuid course_id FK "enforced, cascade"
        text question_text
        enum type "multiple_choice|true_false|short_answer"
        string points
    }
    CHOICES {
        uuid id PK
        uuid question_id FK "conventional"
        string choice_text
        boolean is_correct
    }
    USER_ANSWERS {
        uuid id PK
        bigint user_id FK "enforced, cascade"
        uuid question_id FK "conventional"
        uuid choice_id FK "conventional"
        boolean is_correct
        int attempt_number
    }
    LEADER_BOARDS {
        uuid id PK
        bigint user_id FK "enforced, cascade"
        uuid question_id FK "conventional"
        string score
        int attempt_number
    }
    EXTRA_SCORES {
        uuid id PK
        bigint user_id FK "conventional"
        double score
        text reason "nullable"
    }
    COURSE_PROGRESS {
        uuid id PK
        uuid course_id FK "conventional"
        bigint user_id FK "conventional"
        decimal percentage_watched
        decimal last_watched_time
    }
    COHORTS {
        uuid id PK
        string name
        text cohort_description "nullable"
        datetime start_date "nullable"
        datetime end_date "nullable"
    }
    STUDENTS {
        uuid id PK
        string full_name
        string email
        uuid cohort_id FK "conventional"
        enum status "not-enrolled|enrolled"
    }
```

## Key tables

| Table | Purpose | Notes |
|---|---|---|
| `courses` | Video/audio courses with points and category | Transcription columns added later; `status` supports archive/restore |
| `questions` / `choices` / `answers` | Quiz bank | `questions.course_id` is one of only four enforced FKs; choices and answers are conventional |
| `user_answers` | A learner's answer per question per attempt | `user_id` enforced; question/choice conventional |
| `leader_boards` | Per-question scores feeding the leaderboard | `user_id` enforced; leaderboard sums latest attempt + extra scores |
| `extra_scores` | Manually awarded points | Admin adds/removes via `students/add-points` and `remove-extra-point` |
| `course_progress` | Watch progress per user per course | `percentage_watched >= 90` counts as completed |
| `cohorts` / `students` | Cohort management and pre-enrolment rosters | `students` are invite-stage records, distinct from enrolled `users` |
| `student_migration_logs` | Audit of cohort migrations | All three FKs are commented out — conventional only |

## Leaderboard computation

The public top-students query sums each learner's **latest attempt** scores in `leader_boards`, adds `extra_scores`, counts courses with `percentage_watched >= 90` as `completed_courses`, and filters to `users.status = active`.
