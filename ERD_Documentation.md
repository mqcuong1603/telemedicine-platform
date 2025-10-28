# Entity-Relationship Diagram (ERD) Documentation

## Telemedicine Appointment & Consultation Platform

---

## Overview

This document provides complete documentation for the database schema of the Telemedicine Appointment & Consultation Platform. The design follows normalization principles while maintaining simplicity and practicality for a single-developer, 8-week project.

**Total Tables:** 12  
**Database Type:** Relational (PostgreSQL 14+ or MySQL 8+)  
**Normalization Level:** 3NF (Third Normal Form)  
**Expected Records (Year 1):** ~10,000-50,000

---

## Database Statistics

| Category            | Tables | Purpose                      |
| ------------------- | ------ | ---------------------------- |
| User Management     | 4      | Authentication and profiles  |
| Appointment System  | 2      | Scheduling and consultations |
| Prescription System | 3      | Electronic prescriptions     |
| Medical History     | 1      | Patient medical records      |
| Reviews & Ratings   | 1      | Doctor feedback              |
| Notifications       | 1      | Email/SMS messaging          |
| **Total**           | **12** | Complete system              |

---

## Table-by-Table Documentation

### 1. users (Authentication & Authorization)

**Purpose:** Core authentication table for all system users.

**Key Fields:**

- `email` - Unique login identifier (also used for notifications)
- `password_hash` - Bcrypt encrypted password (never store plain text)
- `role` - User type: 'patient', 'doctor', or 'admin'
- `is_verified` - Email verification status (must be true to use system)
- `verification_token` - Token sent in verification email
- `reset_token` - Token for password reset (expires after use)
- `reset_token_expiry` - When reset token becomes invalid

**Business Rules:**

1. Email must be unique across all roles
2. Password must be hashed with bcrypt (minimum 10 rounds)
3. Users cannot login until `is_verified = true`
4. Reset tokens expire after 1 hour
5. Track `last_login` for security audits

**Security Notes:**

- Never store plain text passwords
- Implement rate limiting on login attempts (5 attempts per 15 min)
- Reset tokens should be cryptographically random
- JWT tokens should expire after 24 hours

**Sample Data:**

```sql
INSERT INTO users (email, password_hash, role, is_verified)
VALUES
  ('john.doe@example.com', '$2b$10$...', 'patient', true),
  ('dr.smith@hospital.com', '$2b$10$...', 'doctor', true),
  ('admin@platform.com', '$2b$10$...', 'admin', true);
```

---

### 2. patients (Patient Demographics)

**Purpose:** Store patient-specific demographic and contact information.

**Key Fields:**

- `user_id` - Foreign key to users table (1-to-1 relationship)
- `date_of_birth` - Required for age calculation and pediatric care
- `gender` - Optional: male, female, other
- `phone` - Primary contact number
- `address`, `city`, `country` - Location information
- `profile_picture` - Path to stored image

**Business Rules:**

1. One user can only have one patient profile
2. Age must be at least 1 year (calculate from DOB)
3. Phone number should be in international format
4. Profile picture stored in cloud storage (URL saved here)

**Calculated Fields (not stored):**

- Age: `YEAR(CURRENT_DATE) - YEAR(date_of_birth)`
- Full Name: `CONCAT(first_name, ' ', last_name)`

**Privacy Considerations:**

- All patient data is HIPAA-sensitive
- Soft delete recommended (keep records for legal retention)
- Access restricted to assigned doctors only

**Sample Data:**

```sql
INSERT INTO patients (user_id, first_name, last_name, date_of_birth, phone)
VALUES
  (1, 'John', 'Doe', '1990-05-15', '+1-555-0123');
```

---

### 3. doctors (Doctor Professional Information)

**Purpose:** Store doctor credentials, specialization, and professional details.

**Key Fields:**

- `user_id` - Foreign key to users table (1-to-1)
- `specialization` - Medical specialty (Cardiology, Dermatology, General Medicine, etc.)
- `qualification` - Educational degrees (MBBS, MD, PhD, etc.)
- `experience_years` - Years of medical practice
- `consultation_fee` - Base fee per consultation
- `bio` - Professional description (max 500 words)
- `rating` - Average patient rating (0-5, calculated)
- `total_reviews` - Count of reviews received
- `is_available` - Can accept new appointments

**Business Rules:**

1. Consultation fee cannot be negative
2. Experience years must be ≥ 0 and ≤ 60
3. Rating calculated as AVG of all review ratings
4. Total_reviews updated via database trigger
5. New doctors start with rating = 0, is_available = true

**Rating Calculation:**

```sql
-- Trigger to update doctor rating after new review
UPDATE doctors
SET rating = (SELECT AVG(rating) FROM reviews WHERE doctor_id = NEW.doctor_id),
    total_reviews = (SELECT COUNT(*) FROM reviews WHERE doctor_id = NEW.doctor_id)
WHERE id = NEW.doctor_id;
```

**Search Optimization:**

- Index on (specialization, rating) for filtered searches
- Full-text search on bio for keyword matching

**Sample Data:**

```sql
INSERT INTO doctors (user_id, first_name, last_name, specialization, qualification, experience_years, consultation_fee)
VALUES
  (2, 'Dr. Jane', 'Smith', 'Cardiology', 'MBBS, MD (Cardiology)', 15, 100.00);
```

---

### 4. admins (Administrator Accounts)

**Purpose:** Store administrator account information.

**Key Fields:**

- `user_id` - Foreign key to users table (1-to-1)
- `first_name`, `last_name` - Admin name
- `phone` - Contact number

**Business Rules:**

1. Admins have elevated system access
2. Cannot book appointments or create prescriptions
3. Can view all data but cannot modify medical records
4. All admin actions should be logged (implement audit log if needed)

**Security:**

- Limit number of admin accounts
- Require strong authentication (consider 2FA)
- Monitor admin activity

**Sample Data:**

```sql
INSERT INTO admins (user_id, first_name, last_name, phone)
VALUES
  (3, 'System', 'Administrator', '+1-555-9999');
```

---

### 5. appointments (Core Appointment & Video Consultation)

**Purpose:** Central table for appointment scheduling, video consultations, and payments.

**Key Fields:**

**Scheduling:**

- `appointment_date`, `start_time`, `end_time` - When appointment occurs
- `status` - Workflow state: pending, confirmed, completed, cancelled
- `reason_for_visit` - Why patient is seeking consultation
- `symptoms` - Optional symptom description

**Video Consultation:**

- `video_room_id` - Unique room identifier (generated from Jitsi/Twilio)
- `video_room_url` - Full URL to join video call
- `patient_joined_at`, `doctor_joined_at` - Attendance tracking
- `consultation_notes` - Doctor's notes from the consultation

**Payment:**

- `payment_amount` - Consultation fee charged
- `payment_status` - unpaid, paid, refunded
- `payment_method` - card, paypal, etc.
- `transaction_id` - Stripe transaction reference
- `paid_at` - Payment timestamp

**Business Rules:**

1. **Double Booking Prevention:**

   ```sql
   UNIQUE INDEX (doctor_id, appointment_date, start_time)
   ```

   Ensures doctor cannot have two appointments at same time

2. **Status Workflow:**

   ```text
   pending → confirmed → completed
      ↓           ↓
   cancelled  cancelled
   ```

3. **Video Room Creation:**

   - Room created when status changes to 'confirmed'
   - Room valid for appointment_date only
   - Accessible 5 minutes before start_time

4. **Payment Rules:**
   - Payment required before confirmation (configurable)
   - Refund processed if cancelled > 24 hours before
   - No refund if cancelled < 24 hours before

**Critical Validations:**

- appointment_date must be in future
- end_time must be after start_time
- start_time must be on 30-minute boundaries (00, 30)
- Cannot cancel completed appointments
- Cannot modify past appointments

**Performance Notes:**

- Index on (doctor_id, appointment_date) for doctor's schedule
- Index on (patient_id, appointment_date) for patient's appointments
- Index on status for filtering
- Index on transaction_id for payment lookups

**Sample Data:**

```sql
INSERT INTO appointments (patient_id, doctor_id, appointment_date, start_time, end_time, reason_for_visit, payment_amount, status)
VALUES
  (1, 1, '2025-11-01', '10:00:00', '10:30:00', 'Annual checkup', 100.00, 'confirmed');
```

---

### 6. doctor_availability (Weekly Schedule)

**Purpose:** Define doctor's recurring weekly availability schedule.

**Key Fields:**

- `doctor_id` - Which doctor this schedule belongs to
- `day_of_week` - 0 = Sunday, 1 = Monday, ..., 6 = Saturday
- `start_time`, `end_time` - Working hours for that day
- `slot_duration` - Length of each appointment slot (30 or 60 minutes)
- `is_available` - Can temporarily disable without deleting

**Business Rules:**

1. Doctors can have multiple time blocks per day

   ```text
   Example for Monday:
   - 09:00-12:00 (Morning session)
   - 14:00-17:00 (Afternoon session)
   ```

2. Slot Generation Algorithm:

   ```python
   slots = []
   current = start_time
   while current + slot_duration <= end_time:
       slots.append(current)
       current += slot_duration
   return slots
   ```

3. Validation:
   - end_time must be after start_time
   - slot_duration must be 30 or 60 minutes
   - No overlapping time blocks for same day

**Example Schedule:**

```sql
-- Dr. Smith available Mon-Fri, 9 AM - 5 PM with 30-min slots
INSERT INTO doctor_availability (doctor_id, day_of_week, start_time, end_time, slot_duration)
VALUES
  (1, 1, '09:00', '12:00', 30),  -- Monday morning
  (1, 1, '14:00', '17:00', 30),  -- Monday afternoon
  (1, 2, '09:00', '12:00', 30),  -- Tuesday morning
  (1, 2, '14:00', '17:00', 30),  -- Tuesday afternoon
  -- ... repeat for Wed, Thu, Fri
```

**Checking Availability:**

```sql
-- Get available slots for a doctor on a specific date
SELECT generate_series(start_time, end_time - slot_duration * INTERVAL '1 minute',
                       slot_duration * INTERVAL '1 minute') AS slot
FROM doctor_availability
WHERE doctor_id = ?
  AND day_of_week = EXTRACT(DOW FROM ?)
  AND is_available = true
  AND slot NOT IN (
    SELECT start_time FROM appointments
    WHERE doctor_id = ? AND appointment_date = ? AND status != 'cancelled'
  );
```

---

### 7. prescriptions (Prescription Header)

**Purpose:** Main prescription record linking appointment to medications.

**Key Fields:**

- `appointment_id` - Which consultation this prescription came from
- `patient_id`, `doctor_id` - Who was involved
- `prescription_date` - When prescription was issued
- `diagnosis` - Medical diagnosis from consultation
- `notes` - General instructions (e.g., "Drink plenty of water", "Avoid dairy")
- `pdf_url` - Link to generated PDF prescription

**Business Rules:**

1. One appointment can have multiple prescriptions (if follow-up)
2. Prescription must be linked to completed appointment
3. PDF generated asynchronously after creation
4. Patient notified via email when prescription ready

**PDF Generation:**

- Use library like pdfkit, jsPDF, or Puppeteer
- Include: Doctor info, Patient info, Date, Diagnosis, Medications table
- Digital signature (image or text)
- Store in cloud storage (AWS S3, Cloudinary)
- URL saved in pdf_url field

**Sample Data:**

```sql
INSERT INTO prescriptions (appointment_id, patient_id, doctor_id, diagnosis, notes)
VALUES
  (1, 1, 1, 'Upper Respiratory Infection', 'Complete full course of antibiotics. Rest and stay hydrated.');
```

---

### 8. prescription_items (Individual Medications)

**Purpose:** Line items listing each medication in a prescription.

**Key Fields:**

- `prescription_id` - Parent prescription
- `medication_id` - Which medication from master list
- `dosage` - Amount per dose (e.g., "500mg", "2 tablets", "10ml")
- `frequency` - How often (e.g., "Twice daily", "Every 8 hours", "Once at night")
- `duration` - How long (e.g., "7 days", "2 weeks", "Until symptoms resolve")
- `instructions` - Special instructions (e.g., "Take with food", "Avoid alcohol")

**Business Rules:**

1. Each prescription can have 1-10 medications typically
2. Dosage format should be validated
3. Frequency options should be standardized:
   - Once daily
   - Twice daily
   - Three times daily
   - Four times daily
   - Every X hours
   - As needed (PRN)

**Quantity Calculation (optional):**

```text
If frequency = "Twice daily" and duration = "7 days"
Then quantity = 2 * 7 = 14 tablets
```

**Sample Data:**

```sql
INSERT INTO prescription_items (prescription_id, medication_id, dosage, frequency, duration, instructions)
VALUES
  (1, 5, '500mg', 'Three times daily', '7 days', 'Take with food'),
  (1, 12, '400mg', 'As needed', '5 days', 'For fever or pain, maximum 4 times daily');
```

---

### 9. medications (Master Drug Database)

**Purpose:** Central repository of available medications.

**Key Fields:**

- `name` - Brand or common medication name
- `generic_name` - Scientific/generic name
- `category` - Drug classification (Antibiotic, Painkiller, etc.)
- `form` - How it's administered (Tablet, Capsule, Syrup, Injection)
- `strength` - Standard dosage (e.g., "500mg", "250mg/5ml")
- `common_interactions` - Brief warning text about major interactions
- `is_active` - Whether still available for prescription

**Categories:**

- Antibiotic
- Antiviral
- Painkiller (Analgesic)
- Anti-inflammatory (NSAID)
- Antacid
- Antihistamine
- Antihypertensive
- Antidiabetic
- Vitamin/Supplement

**Pre-Population:**
This table should be seeded with ~100 common medications:

```sql
INSERT INTO medications (name, generic_name, category, form, strength, common_interactions)
VALUES
  ('Paracetamol', 'Acetaminophen', 'Painkiller', 'Tablet', '500mg', 'Avoid alcohol; do not exceed 4g/day'),
  ('Amoxicillin', 'Amoxicillin', 'Antibiotic', 'Capsule', '500mg', 'Take full course; inform if allergic to penicillin'),
  ('Ibuprofen', 'Ibuprofen', 'Painkiller', 'Tablet', '400mg', 'Take with food; may increase bleeding risk with aspirin'),
  ('Omeprazole', 'Omeprazole', 'Antacid', 'Capsule', '20mg', 'Take before meals'),
  ('Cetirizine', 'Cetirizine', 'Antihistamine', 'Tablet', '10mg', 'May cause drowsiness');
-- ... continue for ~95 more common medications
```

**Interaction Warnings:**

- Display `common_interactions` text when doctor selects medication
- Basic warning only (not comprehensive drug interaction checker)
- Doctor makes final clinical decision

---

### 10. medical_history (Patient Medical Records)

**Purpose:** Store patient's medical conditions, allergies, and health history.

**Key Fields:**

- `patient_id` - Which patient this record belongs to
- `condition_name` - Name of condition or allergy
- `condition_type` - Category:
  - chronic_condition (Diabetes, Hypertension)
  - allergy (Penicillin, Peanuts, Latex)
  - past_surgery (Appendectomy, etc.)
  - family_history (Heart disease in family)
- `diagnosed_date` - When condition was identified
- `status` - active, resolved, managed
- `severity` - For allergies: mild, moderate, severe, life-threatening
- `notes` - Additional details

**Business Rules:**

1. Allergies must be checked before prescribing medications
2. Chronic conditions inform treatment decisions
3. Family history useful for risk assessment
4. Doctors should review before each consultation

**Critical for Safety:**

```sql
-- Check for allergies before prescribing
SELECT condition_name, severity
FROM medical_history
WHERE patient_id = ?
  AND condition_type = 'allergy'
  AND status = 'active';

-- Display warning if prescribing medication patient is allergic to
```

**Sample Data:**

```sql
INSERT INTO medical_history (patient_id, condition_name, condition_type, diagnosed_date, status, severity)
VALUES
  (1, 'Type 2 Diabetes', 'chronic_condition', '2020-03-15', 'managed', null),
  (1, 'Penicillin', 'allergy', '2010-06-20', 'active', 'severe'),
  (1, 'Appendectomy', 'past_surgery', '2015-08-10', 'resolved', null);
```

---

### 11. reviews (Doctor Ratings & Patient Feedback)

**Purpose:** Collect patient feedback about doctors after consultations.

**Key Fields:**

- `appointment_id` - Which consultation being reviewed (1-to-1 relationship)
- `patient_id`, `doctor_id` - Who reviewed whom
- `rating` - Overall rating (1-5 stars, required)
- `review_text` - Optional written review
- `professionalism_rating` - Specific rating for professionalism (1-5)
- `communication_rating` - Specific rating for communication (1-5)
- `is_anonymous` - Hide patient name on public display

**Business Rules:**

1. Only one review per appointment
2. Can only review after appointment status = 'completed'
3. Rating must be 1-5 (validate)
4. Review text optional but recommended
5. Updates doctor's average rating automatically

**Rating Trigger:**

```sql
CREATE TRIGGER update_doctor_rating
AFTER INSERT ON reviews
FOR EACH ROW
BEGIN
  UPDATE doctors
  SET rating = (SELECT AVG(rating) FROM reviews WHERE doctor_id = NEW.doctor_id),
      total_reviews = (SELECT COUNT(*) FROM reviews WHERE doctor_id = NEW.doctor_id)
  WHERE id = NEW.doctor_id;
END;
```

**Display Rules:**

- If `is_anonymous = true`, show "Anonymous Patient" instead of name
- Always show "Verified Patient" badge
- Sort by most recent first
- Show average of all three ratings

**Sample Data:**

```sql
INSERT INTO reviews (appointment_id, patient_id, doctor_id, rating, review_text, professionalism_rating, communication_rating)
VALUES
  (1, 1, 1, 5, 'Excellent doctor! Very thorough and caring. Highly recommend.', 5, 5);
```

---

### 12. notifications (Communication Log)

**Purpose:** Track all email and SMS notifications sent by the system.

**Key Fields:**

- `user_id` - Recipient of notification
- `type` - Category of notification:
  - appointment_confirmation
  - appointment_reminder (24h before)
  - appointment_cancelled
  - prescription_ready
  - payment_success
- `subject` - Email subject line
- `message` - Full notification text
- `delivery_method` - email, sms, or both
- `is_sent` - Whether successfully delivered
- `sent_at` - Delivery timestamp

**Business Rules:**

1. Notifications sent asynchronously (queue-based)
2. Retry failed deliveries up to 3 times
3. Mark is_sent = true only after confirmation from email/SMS service
4. Store for audit trail and debugging

**Notification Triggers:**

- **Appointment Booked:** Immediate confirmation to both patient and doctor
- **Appointment Reminder:** Sent 24 hours before appointment
- **Appointment Cancelled:** Immediate notification to other party
- **Prescription Ready:** When PDF is generated and available
- **Payment Success:** Immediate receipt after payment

**Implementation with Queue:**

```javascript
// Pseudo-code for sending notifications
async function sendNotification(userId, type, data) {
  const notification = await Notification.create({
    user_id: userId,
    type: type,
    subject: generateSubject(type, data),
    message: generateMessage(type, data),
    delivery_method: "email",
  });

  // Add to queue for async processing
  await notificationQueue.add({
    notificationId: notification.id,
    email: user.email,
    subject: notification.subject,
    message: notification.message,
  });
}
```

**Sample Data:**

```sql
INSERT INTO notifications (user_id, type, subject, message, delivery_method, is_sent, sent_at)
VALUES
  (1, 'appointment_confirmation',
   'Appointment Confirmed with Dr. Smith',
   'Your appointment has been confirmed for November 1, 2025 at 10:00 AM...',
   'email', true, '2025-10-25 14:30:00');
```

---

## Entity Relationships

### One-to-One Relationships

```text
users (1) ←→ (1) patients
users (1) ←→ (1) doctors
users (1) ←→ (1) admins
appointments (1) ←→ (1) reviews
```

Each user has exactly one role-specific profile.

### One-to-Many Relationships

```text
patients (1) → (∞) appointments
doctors (1) → (∞) appointments
doctors (1) → (∞) doctor_availability
appointments (1) → (∞) prescriptions
prescriptions (1) → (∞) prescription_items
medications (1) → (∞) prescription_items
patients (1) → (∞) medical_history
users (1) → (∞) notifications
```

### Many-to-Many (through junction tables)

```text
patients (∞) ←→ appointments ←→ (∞) doctors
patients (∞) ←→ medical_history ←→ (∞) conditions
```

---

## Database Indexes

### High Priority (Create First)

```sql
-- Users
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role ON users(role);

-- Appointments (Critical for performance)
CREATE UNIQUE INDEX idx_appointments_no_double_booking
  ON appointments(doctor_id, appointment_date, start_time);
CREATE INDEX idx_appointments_patient ON appointments(patient_id);
CREATE INDEX idx_appointments_doctor_date ON appointments(doctor_id, appointment_date);
CREATE INDEX idx_appointments_status ON appointments(status);

-- Doctors (For search)
CREATE INDEX idx_doctors_specialization ON doctors(specialization);
CREATE INDEX idx_doctors_rating ON doctors(rating);
CREATE INDEX idx_doctors_spec_rating ON doctors(specialization, rating);

-- Prescriptions
CREATE INDEX idx_prescriptions_patient ON prescriptions(patient_id);
CREATE INDEX idx_prescriptions_appointment ON prescriptions(appointment_id);
```

### Medium Priority

```sql
-- Availability
CREATE INDEX idx_doctor_availability_doctor ON doctor_availability(doctor_id);
CREATE INDEX idx_doctor_availability_day ON doctor_availability(doctor_id, day_of_week);

-- Medical History
CREATE INDEX idx_medical_history_patient ON medical_history(patient_id);
CREATE INDEX idx_medical_history_type ON medical_history(patient_id, condition_type);

-- Notifications
CREATE INDEX idx_notifications_user ON notifications(user_id);
CREATE INDEX idx_notifications_sent ON notifications(is_sent);
```

---

## Sample Queries

### 1. Get Available Appointment Slots

```sql
WITH doctor_schedule AS (
  SELECT
    generate_series(
      start_time,
      end_time - (slot_duration || ' minutes')::INTERVAL,
      (slot_duration || ' minutes')::INTERVAL
    ) AS time_slot
  FROM doctor_availability
  WHERE doctor_id = ?
    AND day_of_week = EXTRACT(DOW FROM ?::date)
    AND is_available = true
)
SELECT ds.time_slot
FROM doctor_schedule ds
WHERE NOT EXISTS (
  SELECT 1 FROM appointments a
  WHERE a.doctor_id = ?
    AND a.appointment_date = ?::date
    AND a.start_time = ds.time_slot
    AND a.status NOT IN ('cancelled')
)
ORDER BY ds.time_slot;
```

### 2. Get Patient's Complete Medical Summary

```sql
SELECT
  p.first_name, p.last_name, p.date_of_birth,
  EXTRACT(YEAR FROM AGE(p.date_of_birth)) AS age,
  -- Chronic conditions
  (SELECT JSON_AGG(json_build_object('condition', condition_name, 'status', status))
   FROM medical_history
   WHERE patient_id = p.id AND condition_type = 'chronic_condition') AS chronic_conditions,
  -- Allergies
  (SELECT JSON_AGG(json_build_object('allergen', condition_name, 'severity', severity))
   FROM medical_history
   WHERE patient_id = p.id AND condition_type = 'allergy') AS allergies,
  -- Recent prescriptions
  (SELECT COUNT(*) FROM prescriptions WHERE patient_id = p.id
   AND prescription_date > CURRENT_DATE - INTERVAL '6 months') AS recent_prescriptions
FROM patients p
WHERE p.id = ?;
```

### 3. Doctor's Upcoming Schedule

```sql
SELECT
  a.appointment_date,
  a.start_time,
  a.end_time,
  p.first_name || ' ' || p.last_name AS patient_name,
  a.reason_for_visit,
  a.status
FROM appointments a
JOIN patients p ON a.patient_id = p.id
WHERE a.doctor_id = ?
  AND a.appointment_date >= CURRENT_DATE
  AND a.status IN ('pending', 'confirmed')
ORDER BY a.appointment_date, a.start_time;
```

### 4. Generate Doctor Performance Report

```sql
SELECT
  d.first_name || ' ' || d.last_name AS doctor_name,
  d.specialization,
  COUNT(a.id) AS total_appointments,
  COUNT(CASE WHEN a.status = 'completed' THEN 1 END) AS completed_consultations,
  AVG(r.rating) AS average_rating,
  COUNT(r.id) AS total_reviews,
  SUM(CASE WHEN a.payment_status = 'paid' THEN a.payment_amount ELSE 0 END) AS total_revenue
FROM doctors d
LEFT JOIN appointments a ON d.id = a.doctor_id
LEFT JOIN reviews r ON d.id = r.doctor_id
WHERE a.appointment_date BETWEEN ? AND ?
GROUP BY d.id, d.first_name, d.last_name, d.specialization;
```

---

## Data Migration Plan

### Week 1-2: Core Setup

```sql
-- Phase 1: User management
CREATE TABLE users;
CREATE TABLE patients;
CREATE TABLE doctors;
CREATE TABLE admins;

-- Insert test data
INSERT INTO users, patients, doctors VALUES ...;
```

### Week 3: Appointments

```sql
-- Phase 2: Appointment system
CREATE TABLE appointments;
CREATE TABLE doctor_availability;

-- Seed availability for test doctors
INSERT INTO doctor_availability VALUES ...;
```

### Week 4: Prescriptions

```sql
-- Phase 3: Prescription system
CREATE TABLE prescriptions;
CREATE TABLE prescription_items;
CREATE TABLE medications;

-- Pre-populate medications
INSERT INTO medications VALUES ...;  -- ~100 common drugs
```

### Week 5-6: Supporting Tables

```sql
-- Phase 4: Medical history and reviews
CREATE TABLE medical_history;
CREATE TABLE reviews;
CREATE TABLE notifications;
```

---

## Security Best Practices

### Password Security

- Hash with bcrypt (minimum cost factor 10)
- Never store plain text passwords
- Implement password strength requirements
- Rate limit login attempts

### Data Encryption

- Use HTTPS for all API traffic
- Encrypt sensitive fields at rest (optional but recommended)
- Use parameterized queries to prevent SQL injection

### Access Control

- Implement row-level security where supported
- Patients can only access own records
- Doctors can only access assigned patients
- Admins have read-only access to medical data

### Audit Trail

- Log all authentication attempts
- Log all data modifications (who, what, when)
- Monitor for suspicious activity

---

## Conclusion

This 12-table database design provides a complete, production-ready schema for a telemedicine platform. It balances:

- **Completeness**: All essential features covered
- **Simplicity**: Manageable for single developer
- **Performance**: Properly indexed for common queries
- **Security**: HIPAA-aligned best practices
- **Scalability**: Can handle thousands of users

The design is realistic for an 8-week development timeline while demonstrating professional database design principles.

---

**Document Version:** 1.0  
**Database Version:** PostgreSQL 14+ / MySQL 8+  
**Normalization Level:** 3NF  
**Total Tables:** 12  
**Last Updated:** October 2025
