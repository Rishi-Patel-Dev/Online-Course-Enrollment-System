# Online Course Enrollment System

A robust PostgreSQL database schema designed to handle university course registration, waitlists, and prerequisite management during high-traffic enrollment periods.

## Overview

This system manages the complete lifecycle of course enrollment, from prerequisite validation to automatic waitlist processing. It's built to handle concurrent enrollment attempts, maintain data integrity, and provide real-time enrollment statistics for students and advisors.

## Key Features

- **Concurrent Enrollment Handling**: SERIALIZABLE transactions prevent race conditions when multiple students compete for the last seat
- **Automatic Waitlist Management**: When a student drops a course, the system automatically enrolls the next waitlisted student
- **Prerequisite Enforcement**: Students cannot enroll in courses without completing required prerequisites with passing grades
- **Enrollment History Tracking**: Complete audit trail of all enrollment actions (enrollments, drops, waitlist additions)
- **Real-time Capacity Management**: Automatic updates to course enrollment counts with capacity validation
- **Position Integrity**: Waitlist positions are maintained without gaps or duplicates

## Database Schema

### Core Tables

**Student**
- Stores student information with unique email validation
- Year validation (1-4) for undergraduate students
- Links to all enrollment and waitlist records

**Course**
- Tracks course details, capacity, and current enrollment
- Enforces capacity constraints at the database level
- Self-referencing prerequisites through Prerequisite table

**Enrollment**
- Records active, dropped, and completed course enrollments
- Prevents duplicate active enrollments per student/course
- Tracks grades with validation

**Waitlist**
- Maintains ordered queue of students waiting for course seats
- Unique position numbers per course
- Prevents duplicate waitlist entries

**Prerequisite**
- Self-referencing relationship for course dependencies
- Prevents circular prerequisites and self-prerequisites

**Enrollment_History**
- Immutable audit log of all enrollment actions
- Tracks enrollment, drops, and waitlist additions with timestamps

## Business Logic Implementation

### Triggers

1. **Prerequisite Enforcement** (`check_prerequisites`)
   - Validates prerequisite completion before enrollment
   - Requires passing grades (A, B, C, D) in prerequisite courses
   - Executed BEFORE INSERT on Enrollment

2. **Enrollment Count Management** (`update_enrollment_count`, `handle_drop`)
   - Automatically increments/decrements course enrollment counts
   - Validates capacity limits
   - Maintains consistency between Course.current_enrollment and actual enrollments

3. **Automatic Waitlist Processing** (`auto_enroll_from_waitlist`)
   - Triggers when enrollment count decreases (student drops)
   - Enrolls first waitlisted student automatically
   - Updates remaining waitlist positions

4. **Audit Logging** (`log_enrollment_action`)
   - Records all enrollment and drop actions
   - Provides complete history for compliance and analytics

### Views

**Course_Availability**
- Real-time course capacity and waitlist information
- Calculates available seats
- Aggregates waitlist counts

**Student_Enrollments**
- Active enrollment details per student
- Includes course information and credits
- Filtered to show only current enrollments

**Waitlist_Status**
- Complete waitlist information with student details
- Ordered by course and position
- Shows estimated seats until enrollment

## Transaction Management

### Critical Operations

**Student Enrollment** (SERIALIZABLE)
```sql
-- Prevents race conditions on last seat
-- Locks course row during capacity check
-- Ensures atomic enrollment or waitlist addition
```

**Course Drop with Auto-Enrollment** (SERIALIZABLE)
```sql
-- Atomically drops student and enrolls from waitlist
-- Maintains consistency between counts and actual enrollments
-- Prevents manual enrollment during auto-processing
```

**Batch Enrollment** (READ COMMITTED)
```sql
-- Allows student to enroll in multiple courses
-- Independent validation per course
-- Better concurrency during high traffic
```

**Waitlist Position Updates** (SERIALIZABLE)
```sql
-- Maintains position integrity
-- Prevents gaps or duplicate positions
-- Ensures fair queue processing
```

**Advisor Reports** (READ ONLY + REPEATABLE READ)
```sql
-- Consistent snapshot for reporting
-- Doesn't block concurrent enrollments
-- Optimal performance for analytics
```

## Concurrency & Performance

### Isolation Levels

- **SERIALIZABLE**: Used for operations requiring strict consistency (enrollment in last seat, waitlist processing)
- **READ COMMITTED**: Used for independent operations (multi-course enrollment) to maximize throughput
- **REPEATABLE READ**: Used for reports requiring consistent snapshots without blocking writes

### Locking Strategy

- `FOR UPDATE` locks prevent double-enrollment in last seat
- Trigger-based updates maintain consistency without explicit locks
- History table uses append-only pattern for maximum concurrency

## Use Cases

### High-Traffic Scenarios

**Registration Period Opens**
- Thousands of students enrolling simultaneously
- READ COMMITTED transactions for multi-course enrollments
- Triggers handle capacity validation independently per course

**Popular Course Fills**
- SERIALIZABLE isolation prevents overselling
- Automatic waitlist creation
- Fair processing based on request timestamp

**Mass Course Drops** (e.g., schedule changes)
- Automatic waitlist processing cascades enrollments
- Position updates maintain queue integrity
- History tracking for audit requirements

### Administrative Functions

**Advisor Queries**
- Non-blocking reports via READ ONLY transactions
- Views provide pre-aggregated statistics
- Real-time capacity and waitlist information

**System Maintenance**
- CASCADE delete for student records
- RESTRICT delete for courses (protects historical data)
- Audit trail survives record deletions

## Installation & Setup

### Prerequisites
- PostgreSQL 12 or higher
- PL/pgSQL support enabled

### Deployment Steps

1. Execute schema creation scripts in order:
   - Core tables (Student, Course)
   - Relationship tables (Prerequisite, Enrollment, Waitlist)
   - History table
   
2. Create views for reporting

3. Install trigger functions and triggers

4. Grant appropriate permissions to application users

### Configuration

- Adjust `max_capacity` values per institutional requirements
- Configure semester values for academic calendar
- Set up database connection pooling for high concurrency

## Security Considerations

- Email uniqueness prevents duplicate accounts
- Grade validation ensures data quality
- Foreign key constraints maintain referential integrity
- History table provides tamper-evident audit trail
- CHECK constraints prevent invalid state transitions

## Future Enhancements

- Add instructor assignment and course sections
- Implement enrollment holds (financial, academic)
- Support for waitlist expiration and notifications
- Integration with student information system
- Real-time notification system for waitlist movement

## Author

**Rishi Patel**  

## License

This database schema is provided as an educational resource and reference implementation for course enrollment systems.
