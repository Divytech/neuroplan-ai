# Design Document: AI Study Planner and Smart Notes Generator

## Overview

The AI Study Planner and Smart Notes Generator is a cloud-based learning management system that leverages AI to create personalized study schedules and generate summarized learning materials. The system follows a microservices architecture with separate services for document processing, AI-powered content analysis, schedule generation, and user management.

The core workflow involves:
1. Students upload study materials (PDFs, documents, images)
2. Document processing service extracts and structures content
3. AI analysis service identifies topics, complexity, and generates summaries
4. Schedule generation service creates optimized daily study plans
5. Progress tracking service monitors completion and identifies weak areas
6. Dynamic rescheduling service adjusts plans based on actual progress

The system is designed for high availability and scalability to support thousands of concurrent students during peak exam seasons.

## Architecture

### System Architecture

The system uses a microservices architecture deployed on cloud infrastructure:

```
┌─────────────────────────────────────────────────────────────┐
│                     Client Layer                             │
│  ┌──────────────┐              ┌──────────────┐            │
│  │  Web Client  │              │ Mobile App   │            │
│  │  (React)     │              │ (React Native)│           │
│  └──────────────┘              └──────────────┘            │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                     API Gateway                              │
│              (Authentication, Rate Limiting)                 │
└─────────────────────────────────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│   Document   │   │   Schedule   │   │   Progress   │
│  Processing  │   │  Generation  │   │   Tracking   │
│   Service    │   │   Service    │   │   Service    │
└──────────────┘   └──────────────┘   └──────────────┘
        │                   │                   │
        ▼                   ▼                   ▼
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│  AI Analysis │   │    User      │   │   Storage    │
│   Service    │   │ Management   │   │   Service    │
│  (LLM API)   │   │   Service    │   │   (S3/DB)    │
└──────────────┘   └──────────────┘   └──────────────┘
```

### Technology Stack

- **Frontend**: React (web), React Native (mobile)
- **API Gateway**: Node.js with Express or AWS API Gateway
- **Backend Services**: Python (FastAPI) for AI/ML services, Node.js for business logic
- **AI/ML**: OpenAI API or similar LLM for content analysis and summarization
- **Document Processing**: PyPDF2, python-docx, Tesseract OCR for image text extraction
- **Database**: PostgreSQL for structured data, MongoDB for document storage
- **File Storage**: AWS S3 or equivalent cloud storage
- **Authentication**: JWT tokens with OAuth2 support
- **Message Queue**: Redis or RabbitMQ for async processing
- **Caching**: Redis for session management and frequently accessed data

## Components and Interfaces

### 1. Document Processing Service

**Responsibility**: Extract and structure content from uploaded study materials

**Key Functions**:
- `upload_file(user_id: str, file: bytes, filename: str) -> FileMetadata`
- `extract_text(file_id: str) -> ExtractedContent`
- `parse_syllabus(content: str) -> SyllabusStructure`

**Interfaces**:
```python
class FileMetadata:
    file_id: str
    user_id: str
    filename: str
    file_type: str
    size_bytes: int
    upload_timestamp: datetime
    processing_status: str  # "pending", "processing", "completed", "failed"

class ExtractedContent:
    file_id: str
    raw_text: str
    page_count: int
    word_count: int
    extraction_method: str  # "direct", "ocr"

class SyllabusStructure:
    topics: List[Topic]
    total_estimated_hours: float

class Topic:
    topic_id: str
    name: str
    subtopics: List[str]
    estimated_complexity: int  # 1-5 scale
    keywords: List[str]
```

### 2. AI Analysis Service

**Responsibility**: Analyze content, generate summaries, and estimate study requirements

**Key Functions**:
- `analyze_content(content: str) -> ContentAnalysis`
- `generate_summary(content: str, topic: str) -> Summary`
- `estimate_study_time(topic: Topic, content: str) -> float`

**Interfaces**:
```python
class ContentAnalysis:
    topics: List[Topic]
    key_concepts: List[str]
    difficulty_level: int  # 1-5 scale
    recommended_study_hours: float

class Summary:
    topic_id: str
    original_length: int
    summary_text: str
    summary_length: int
    key_points: List[str]
    formulas: List[str]
    definitions: Dict[str, str]
```

### 3. Schedule Generation Service

**Responsibility**: Create and optimize daily study plans

**Key Functions**:
- `generate_plan(user_id: str, topics: List[Topic], exam_date: date, daily_hours: float) -> StudyPlan`
- `optimize_schedule(plan: StudyPlan, constraints: Constraints) -> StudyPlan`
- `reschedule(plan_id: str, missed_sessions: List[str]) -> StudyPlan`

**Interfaces**:
```python
class StudyPlan:
    plan_id: str
    user_id: str
    exam_date: date
    daily_study_hours: float
    sessions: List[StudySession]
    total_topics: int
    created_at: datetime
    last_updated: datetime

class StudySession:
    session_id: str
    date: date
    topic_id: str
    topic_name: str
    duration_hours: float
    content_references: List[str]
    status: str  # "pending", "completed", "missed", "partial"
    completion_timestamp: Optional[datetime]

class Constraints:
    max_daily_hours: float
    min_session_duration: float  # minimum 0.5 hours
    buffer_percentage: float  # default 20% for revision
    prioritize_weak_topics: bool
```

### 4. Progress Tracking Service

**Responsibility**: Monitor student progress and identify weak topics

**Key Functions**:
- `record_completion(session_id: str, rating: int, notes: str) -> None`
- `get_progress(user_id: str, plan_id: str) -> ProgressReport`
- `identify_weak_topics(user_id: str, plan_id: str) -> List[WeakTopic]`

**Interfaces**:
```python
class ProgressReport:
    plan_id: str
    total_sessions: int
    completed_sessions: int
    missed_sessions: int
    completion_percentage: float
    days_remaining: int
    weak_topics: List[WeakTopic]
    study_time_by_topic: Dict[str, float]
    adherence_score: float  # 0-100

class WeakTopic:
    topic_id: str
    topic_name: str
    average_rating: float
    attempts: int
    additional_time_needed: float
    priority: int  # 1-5, higher is more urgent

class SessionCompletion:
    session_id: str
    user_id: str
    completion_status: str  # "full", "partial", "skipped"
    understanding_rating: int  # 1-5
    time_spent: float
    notes: str
    timestamp: datetime
```

### 5. User Management Service

**Responsibility**: Handle authentication, authorization, and user data

**Key Functions**:
- `register_user(email: str, password: str) -> User`
- `authenticate(email: str, password: str) -> AuthToken`
- `verify_email(token: str) -> bool`
- `get_user_plans(user_id: str) -> List[StudyPlan]`

**Interfaces**:
```python
class User:
    user_id: str
    email: str
    password_hash: str
    email_verified: bool
    created_at: datetime
    last_login: datetime
    preferences: UserPreferences

class UserPreferences:
    timezone: str
    notification_enabled: bool
    default_daily_hours: float
    theme: str  # "light", "dark"

class AuthToken:
    token: str
    user_id: str
    expires_at: datetime
    refresh_token: str
```

### 6. Notes Generator Service

**Responsibility**: Create and format revision notes

**Key Functions**:
- `generate_notes(topic_id: str, content: str) -> RevisionNotes`
- `format_notes(notes: RevisionNotes, format: str) -> bytes`
- `get_notes(user_id: str, topic_id: str) -> RevisionNotes`

**Interfaces**:
```python
class RevisionNotes:
    notes_id: str
    topic_id: str
    user_id: str
    content: str  # Markdown format
    key_points: List[str]
    formulas: List[str]
    definitions: Dict[str, str]
    created_at: datetime
    format_version: str

class NotesExport:
    notes_id: str
    format: str  # "pdf", "markdown", "html"
    file_data: bytes
    filename: str
```

## Data Models

### Database Schema

**Users Table**:
```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    email_verified BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_login TIMESTAMP,
    preferences JSONB
);
```

**Study Plans Table**:
```sql
CREATE TABLE study_plans (
    plan_id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(user_id) ON DELETE CASCADE,
    exam_date DATE NOT NULL,
    daily_study_hours FLOAT NOT NULL,
    total_topics INT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status VARCHAR(50) DEFAULT 'active'
);
```

**Study Sessions Table**:
```sql
CREATE TABLE study_sessions (
    session_id UUID PRIMARY KEY,
    plan_id UUID REFERENCES study_plans(plan_id) ON DELETE CASCADE,
    date DATE NOT NULL,
    topic_id UUID NOT NULL,
    topic_name VARCHAR(255) NOT NULL,
    duration_hours FLOAT NOT NULL,
    status VARCHAR(50) DEFAULT 'pending',
    understanding_rating INT CHECK (understanding_rating BETWEEN 1 AND 5),
    completion_timestamp TIMESTAMP,
    time_spent FLOAT,
    notes TEXT
);
```

**Topics Table**:
```sql
CREATE TABLE topics (
    topic_id UUID PRIMARY KEY,
    plan_id UUID REFERENCES study_plans(plan_id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    subtopics JSONB,
    estimated_complexity INT CHECK (estimated_complexity BETWEEN 1 AND 5),
    keywords JSONB,
    estimated_hours FLOAT
);
```

**Files Table**:
```sql
CREATE TABLE files (
    file_id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(user_id) ON DELETE CASCADE,
    filename VARCHAR(255) NOT NULL,
    file_type VARCHAR(50) NOT NULL,
    size_bytes BIGINT NOT NULL,
    storage_path VARCHAR(500) NOT NULL,
    upload_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    processing_status VARCHAR(50) DEFAULT 'pending'
);
```

**Revision Notes Table**:
```sql
CREATE TABLE revision_notes (
    notes_id UUID PRIMARY KEY,
    topic_id UUID REFERENCES topics(topic_id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(user_id) ON DELETE CASCADE,
    content TEXT NOT NULL,
    key_points JSONB,
    formulas JSONB,
    definitions JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Data Flow

1. **Upload Flow**: Client → API Gateway → Document Processing Service → Storage Service → AI Analysis Service
2. **Plan Generation Flow**: Client → API Gateway → Schedule Generation Service → Database
3. **Progress Update Flow**: Client → API Gateway → Progress Tracking Service → Database → Schedule Generation Service (if rescheduling needed)
4. **Notes Retrieval Flow**: Client → API Gateway → Notes Generator Service → Cache/Database → Client



## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: File Type Validation

*For any* uploaded file, the system should accept the file if and only if its format is one of PDF, DOCX, TXT, JPG, or PNG, and reject all other formats with an appropriate error message.

**Validates: Requirements 1.1**

### Property 2: Error Message Descriptiveness

*For any* operation that fails (file extraction, AI processing, validation), the system should return an error message that includes the specific operation that failed and actionable guidance for the user.

**Validates: Requirements 1.3, 10.2**

### Property 3: Multi-File Content Combination

*For any* set of uploaded files, the combined study material should contain all extractable content from each individual file, preserving the order of upload.

**Validates: Requirements 1.4**

### Property 4: Syllabus Structure Extraction

*For any* syllabus document with structured content, the system should extract all topic names, subtopics, and learning objectives that follow standard syllabus formatting patterns.

**Validates: Requirements 1.5**

### Property 5: Future Date Validation

*For any* date input for exam date, the system should accept the date if and only if it is strictly after the current date.

**Validates: Requirements 2.1**

### Property 6: Study Hours Range Validation

*For any* numeric input for daily study hours, the system should accept the value if and only if it falls within the range [0.5, 16.0] inclusive.

**Validates: Requirements 2.2**

### Property 7: Parameter Update Triggers Regeneration

*For any* existing study plan, when exam date or daily study hours are modified, the system should generate a new schedule that differs from the original and reflects the updated constraints.

**Validates: Requirements 2.4**

### Property 8: Available Study Time Calculation

*For any* exam date and daily study hours, the total available study time should equal (days_until_exam - 1) × daily_study_hours, where days_until_exam is calculated from the current date.

**Validates: Requirements 2.5**

### Property 9: Complete Topic Coverage

*For any* set of topics and valid study parameters, the generated study plan should include at least one study session for every topic in the input set.

**Validates: Requirements 3.1**

### Property 10: Daily Hour Limit Invariance

*For any* generated or rescheduled study plan, the sum of session durations on any single day should never exceed the specified daily study hour limit.

**Validates: Requirements 3.2, 6.3**

### Property 11: Complexity-Based Time Allocation

*For any* two topics with different complexity ratings, the topic with higher complexity should be allocated equal or greater total study time than the topic with lower complexity.

**Validates: Requirements 3.3**

### Property 12: Revision Buffer Allocation

*For any* generated study plan, the sessions scheduled in the final 20% of the time period (by date) should be marked as revision sessions or have reduced new content load.

**Validates: Requirements 3.5**

### Property 13: Notes Generation Completeness

*For any* set of processed topics, the system should generate revision notes for each topic, resulting in a one-to-one mapping between topics and note documents.

**Validates: Requirements 4.1**

### Property 14: Notes Content Extraction

*For any* generated revision notes, if the source material contains formulas, definitions, or key concepts, the notes should include structured representations of these elements.

**Validates: Requirements 4.2**

### Property 15: Content Compression Ratio

*For any* generated revision notes, the character count of the summary should be at most 40% of the character count of the original source material.

**Validates: Requirements 4.3**

### Property 16: Notes Formatting Structure

*For any* generated revision notes in markdown format, the content should contain at least one heading element and at least one list element (bullet or numbered).

**Validates: Requirements 4.4**

### Property 17: Multi-Format Export

*For any* revision notes document, the system should successfully export the content in both PDF and markdown formats without data loss.

**Validates: Requirements 4.5**

### Property 18: Weak Topic Classification

*For any* study session completion with an understanding rating, the associated topic should be flagged as weak if and only if the rating is less than 3.

**Validates: Requirements 5.2**

### Property 19: Weak Topic Scheduling Priority

*For any* set of weak topics with different importance levels and exam proximities, the schedule should allocate additional time to weak topics, with higher priority topics receiving time allocation before lower priority topics.

**Validates: Requirements 5.3, 5.4**

### Property 20: Missed Content Redistribution

*For any* missed study session, the content from that session should appear in one or more future sessions, and the total duration allocated to that content should equal or exceed the original session duration.

**Validates: Requirements 6.2**

### Property 21: Partial Completion Rescheduling

*For any* study session marked as partially completed with X% completion, the rescheduled content should represent (100-X)% of the original session duration.

**Validates: Requirements 6.5**

### Property 22: Cross-Device Data Synchronization

*For any* user accessing the system from multiple devices, changes made to study plans, notes, or progress on one device should be reflected on all other devices within a reasonable synchronization window.

**Validates: Requirements 7.3**

### Property 23: Offline Operation and Sync

*For any* operations performed while offline (viewing notes, marking sessions complete), the data should persist locally and synchronize with the server when connectivity is restored, without data loss.

**Validates: Requirements 7.4**

### Property 24: Authentication Credential Validation

*For any* login attempt, the system should grant access if and only if the provided email and password match a verified user account, and should reject all other attempts with appropriate error messages.

**Validates: Requirements 8.2**

### Property 25: Multi-Plan Management

*For any* user, the system should allow creation and independent management of multiple study plans, where operations on one plan do not affect the state of other plans.

**Validates: Requirements 8.5**

### Property 26: Progress Metric Accuracy

*For any* study plan, the completion percentage should equal (completed_sessions / total_sessions) × 100, and should update immediately when session status changes.

**Validates: Requirements 9.1, 9.2**

### Property 27: Progress Report Completeness

*For any* weekly progress report, the report should include lists of completed topics, identified weak areas, and an adherence score calculated from scheduled vs. completed sessions.

**Validates: Requirements 9.3**

### Property 28: Readiness Indicator Calculation

*For any* study plan approaching the exam date, the readiness indicator should be a function of completion percentage and the inverse of the number of unresolved weak topics.

**Validates: Requirements 9.5**

### Property 29: Rate Limiting Enforcement

*For any* user making API requests, the system should allow up to N requests per time window, and should reject subsequent requests with a 429 status code until the window resets.

**Validates: Requirements 10.5**

## Error Handling

### Error Categories

1. **Validation Errors**: Invalid input parameters, unsupported file formats, constraint violations
2. **Processing Errors**: File extraction failures, AI service timeouts, parsing errors
3. **Resource Errors**: Insufficient storage, rate limit exceeded, quota exceeded
4. **Authentication Errors**: Invalid credentials, expired sessions, unverified email
5. **System Errors**: Database connection failures, external service unavailability

### Error Response Format

All errors should follow a consistent JSON structure:

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable error description",
    "details": {
      "field": "specific_field_name",
      "constraint": "constraint_violated"
    },
    "suggested_action": "What the user should do next",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

### Error Handling Strategies

1. **Retry Logic**: Implement exponential backoff for transient failures (network issues, service timeouts)
2. **Graceful Degradation**: If AI summarization fails, provide extracted text without summarization
3. **User Notification**: Display clear, actionable error messages in the UI
4. **Logging**: Log all errors with context (user_id, operation, parameters) for debugging
5. **Fallback Options**: Offer alternative actions when primary operation fails (e.g., manual schedule adjustment if auto-generation fails)

### Specific Error Scenarios

**File Upload Failures**:
- Network interruption: Store partial upload, allow resume
- Invalid format: Reject immediately with supported format list
- File too large: Reject with size limit information
- Extraction failure: Notify user, allow manual text input

**Schedule Generation Failures**:
- Insufficient time: Calculate minimum required daily hours or latest acceptable exam date
- No topics extracted: Prompt user to manually enter topics
- Optimization timeout: Return suboptimal schedule with warning

**AI Service Failures**:
- Timeout: Retry up to 3 times, then fallback to rule-based processing
- Rate limit: Queue request for later processing, notify user of delay
- Invalid response: Log error, use cached results if available

## Testing Strategy

### Dual Testing Approach

The system will employ both unit testing and property-based testing to ensure comprehensive correctness:

- **Unit tests**: Verify specific examples, edge cases, and error conditions
- **Property tests**: Verify universal properties across all inputs using randomized test data

Both testing approaches are complementary and necessary. Unit tests catch concrete bugs in specific scenarios, while property tests verify general correctness across a wide range of inputs.

### Property-Based Testing Configuration

**Framework Selection**:
- **Python services**: Use Hypothesis library for property-based testing
- **JavaScript/TypeScript services**: Use fast-check library for property-based testing

**Test Configuration**:
- Each property test must run a minimum of 100 iterations with randomized inputs
- Each test must include a comment tag referencing the design document property
- Tag format: `# Feature: ai-study-planner, Property {number}: {property_text}`

**Example Property Test Structure** (Python with Hypothesis):

```python
from hypothesis import given, strategies as st
import hypothesis

@given(
    exam_date=st.dates(min_value=date.today() + timedelta(days=1)),
    daily_hours=st.floats(min_value=0.5, max_value=16.0)
)
@hypothesis.settings(max_examples=100)
def test_available_study_time_calculation(exam_date, daily_hours):
    """
    Feature: ai-study-planner, Property 8: Available Study Time Calculation
    For any exam date and daily study hours, the total available study time 
    should equal (days_until_exam - 1) × daily_study_hours
    """
    plan = generate_plan(exam_date=exam_date, daily_hours=daily_hours)
    days_until_exam = (exam_date - date.today()).days
    expected_time = (days_until_exam - 1) * daily_hours
    assert abs(plan.total_available_hours - expected_time) < 0.01
```

### Unit Testing Strategy

**Focus Areas for Unit Tests**:
1. **Specific Examples**: Test known input-output pairs (e.g., specific syllabus formats)
2. **Edge Cases**: Empty files, single-topic plans, exam tomorrow, maximum daily hours
3. **Error Conditions**: Invalid credentials, corrupted files, network failures
4. **Integration Points**: API endpoint contracts, database transactions, external service calls

**Unit Test Balance**:
- Avoid writing excessive unit tests for scenarios covered by property tests
- Focus unit tests on concrete examples that demonstrate correct behavior
- Use unit tests for integration testing between components
- Property tests handle comprehensive input coverage

### Test Coverage Goals

- **Code Coverage**: Minimum 80% line coverage, 70% branch coverage
- **Property Coverage**: Each correctness property must have at least one property-based test
- **API Coverage**: All public API endpoints must have integration tests
- **Error Coverage**: All error codes must be triggered in at least one test

### Testing Environments

1. **Local Development**: Fast unit tests, subset of property tests (10 iterations)
2. **CI/CD Pipeline**: Full unit test suite, full property tests (100 iterations)
3. **Staging**: Integration tests, end-to-end tests, performance tests
4. **Production**: Smoke tests, health checks, monitoring

### Performance Testing

While not part of correctness properties, performance testing should verify:
- File processing completes within 30 seconds for 50MB files
- API response times under 200ms for 95th percentile
- System handles 1000 concurrent users
- Database queries complete within 100ms

### Security Testing

Security testing should verify:
- All data encrypted at rest and in transit
- SQL injection prevention
- XSS prevention in user-generated content
- Rate limiting effectiveness
- Authentication token expiration
