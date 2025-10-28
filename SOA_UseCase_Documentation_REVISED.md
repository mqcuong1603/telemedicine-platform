# Use Case Diagram Documentation

## Telemedicine Appointment & Consultation Platform
### MVP Implementation - 2 Core Functions (8 Weeks)

---

## Overview

This document provides documentation for all use cases in the Telemedicine Appointment & Consultation Platform. Following an **MVP (Minimum Viable Product) approach**, the system prioritizes **2 core functions** for complete implementation in Weeks 1-6, with stretch goals for Weeks 7-8.

**Core Use Cases:** 28 (Must-Have)
**Stretch Use Cases:** 8 (Nice-to-Have)
**Primary Actors:** 3 (Patient, Doctor, Administrator)
**External Systems:** 2 Core (Video, Email) + 1 Optional (Payment)
**Core Functions:** 2 (Appointment Scheduling, Video Consultation)

---

## System Actors

### 1. Patient
End-users seeking medical consultation through the platform.

**Core Responsibilities (MVP):**
- Register and manage account
- Search and book appointments with doctors
- Participate in video consultations
- Rate and review doctors

**Stretch Responsibilities (Week 7-8):**
- Receive electronic prescriptions
- Make payments for consultations
- Maintain medical history

### 2. Doctor
Licensed healthcare professionals providing medical services.

**Core Responsibilities (MVP):**
- Manage profile and availability schedules
- Accept/reject appointment requests
- Conduct video consultations
- Add consultation notes

**Stretch Responsibilities (Week 7-8):**
- Create electronic prescriptions
- View earnings statistics

### 3. Administrator
System administrators managing platform operations.

**Responsibilities (Basic - Week 7-8):**
- View all users
- Manage user accounts (approve, suspend)
- Monitor appointments

---

## Use Case Summary Table

### 🎯 Phase 1: Authentication & Foundation (Weeks 1-2) - 7 Use Cases

| ID   | Use Case Name        | Primary Actor   | Priority | Week  |
| ---- | -------------------- | --------------- | -------- | ----- |
| UC1  | Register Account     | Patient, Doctor | High     | 1-2   |
| UC2  | Login                | All             | High     | 1-2   |
| UC3  | Logout               | All             | High     | 1-2   |
| UC4  | View Profile         | All             | High     | 1-2   |
| UC5  | Edit Profile         | All             | Medium   | 1-2   |
| UC6  | Reset Password       | Patient, Doctor | Medium   | 7-8   |
| UC7  | Verify Email         | System          | High     | 1-2   |

---

### 🎯 Phase 2: Core Function 1 - Appointment Scheduling (Weeks 3-5) - 10 Use Cases

| ID    | Use Case Name             | Primary Actor   | Priority     | Week |
| ----- | ------------------------- | --------------- | ------------ | ---- |
| UC8   | Browse Doctors            | Patient         | **Critical** | 3-4  |
| UC9   | Search by Specialization  | Patient         | **Critical** | 3-4  |
| UC10  | View Doctor Profile       | Patient         | **Critical** | 3-4  |
| UC11  | Book Appointment          | Patient         | **Critical** | 3-4  |
| UC12  | View My Appointments      | Patient         | High         | 3-4  |
| UC13  | Cancel Appointment        | Patient         | High         | 5    |
| UC14  | Manage Availability       | Doctor          | **Critical** | 3-4  |
| UC15  | View Appointment Requests | Doctor          | **Critical** | 3-4  |
| UC16  | Accept/Reject Appointment | Doctor          | High         | 3-4  |
| UC17  | Check Availability        | System          | **Critical** | 3-4  |

---

### 🎯 Phase 3: Core Function 2 - Video Consultation (Week 6) - 6 Use Cases

| ID   | Use Case Name          | Primary Actor | Priority     | Week |
| ---- | ---------------------- | ------------- | ------------ | ---- |
| UC18 | Join Video Consultation| Patient       | **Critical** | 6    |
| UC19 | Start Video Consultation| Doctor       | **Critical** | 6    |
| UC20 | Add Consultation Notes | Doctor        | High         | 6    |
| UC21 | View Patient Details   | Doctor        | High         | 6    |
| UC22 | Generate Video Room    | System        | **Critical** | 6    |
| UC23 | End Consultation       | Doctor        | High         | 6    |

---

### 🎯 Phase 4: Reviews & Notifications (Week 5) - 5 Use Cases

| ID   | Use Case Name         | Primary Actor | Priority | Week |
| ---- | --------------------- | ------------- | -------- | ---- |
| UC24 | Rate Doctor           | Patient       | High     | 5    |
| UC25 | Write Review          | Doctor        | Medium   | 5    |
| UC26 | View My Reviews       | Doctor        | Medium   | 5    |
| UC27 | Send Notifications    | System        | High     | 3-5  |
| UC28 | Validate Credentials  | System        | High     | 1-2  |

---

### ⭐ Stretch Goals (Week 7-8) - 8 Use Cases

| ID   | Use Case Name             | Primary Actor | Priority | Week |
| ---- | ------------------------- | ------------- | -------- | ---- |
| UC29 | Create Prescription       | Doctor        | Medium   | 7    |
| UC30 | View Prescriptions        | Patient       | Medium   | 7    |
| UC31 | Download Prescription PDF | Patient       | Low      | 7-8  |
| UC32 | Make Payment              | Patient       | Medium   | 7    |
| UC33 | Process Payment           | System        | Medium   | 7    |
| UC34 | View Medical History      | Patient       | Low      | 7-8  |
| UC35 | Update Medical History    | Patient       | Low      | 7-8  |
| UC36 | Admin Dashboard           | Admin         | Low      | 7-8  |

**Total Use Cases: 28 Core + 8 Stretch = 36 (Reduced from 41)**

---

## Core Function 1: Intelligent Appointment Scheduling

**Implementation: Weeks 3-5 | Priority: CRITICAL**

### Workflow

```
Patient Journey:
  Browse Doctors (UC8) → Search by Specialty (UC9) → View Profile (UC10)
  → Book Appointment (UC11) → System Checks Availability (UC17)
  → Email Notification Sent (UC27) → Confirmed

Doctor Journey:
  Manage Availability (UC14) → View Requests (UC15)
  → Accept/Reject (UC16) → Email Notification Sent (UC27)
```

### Key Features

- **Real-time Availability Checking**: Doctor's schedule queried live to prevent conflicts
- **Double Booking Prevention**: Database unique constraint on (doctor_id, date, time)
- **Automated Email Confirmations**: Sent to both patient and doctor
- **Cancellation Support**: Patients can cancel with email notifications

### Success Criteria

- ✅ Patient can search/filter doctors by specialization
- ✅ Patient can view available time slots
- ✅ Patient can book appointment with confirmation email
- ✅ No double bookings possible (system enforced)
- ✅ Doctor can set weekly availability
- ✅ Doctor can view and manage requests
- ✅ Cancellation sends notifications to both parties

### Database Tables Involved

- `users`, `patients`, `doctors`
- `appointments` (status: pending, confirmed, cancelled)
- `doctor_availability` (weekly schedule)
- `notifications` (email tracking)

---

## Core Function 2: Video Consultation System

**Implementation: Week 6 | Priority: CRITICAL**

### Workflow

```
Pre-Consultation:
  System generates unique video room (UC22)
  → Room URL stored in appointments table

Patient Side:
  Join Video (UC18) → Enter waiting room → Doctor admits → Video call

Doctor Side:
  Start Consultation (UC19) → Add Notes (UC20) → View Patient Info (UC21)
  → End Consultation (UC23)
```

### Key Features

- **Jitsi Meet Integration**: Simple iframe embed approach (easiest)
- **Unique Room Per Appointment**: `video_room_id` generated, stored in DB
- **Waiting Room**: Patients wait until doctor joins
- **Consultation Notes**: Doctors can add notes during/after call
- **Session Tracking**: Start/end timestamps recorded

### Success Criteria

- ✅ Unique video room generated for each appointment
- ✅ Both patient and doctor can join the call
- ✅ HD video and audio quality maintained
- ✅ Doctor can add consultation notes
- ✅ Session duration tracked

### Technical Implementation

```javascript
// Simple Jitsi Meet Embed Example
const domain = 'meet.jit.si';
const roomName = `appointment_${appointmentId}_${timestamp}`;
const options = {
  roomName: roomName,
  width: '100%',
  height: 700,
  parentNode: document.querySelector('#jitsi-container')
};
const api = new JitsiMeetExternalAPI(domain, options);
```

### Database Tables Involved

- `appointments.video_room_id`
- `appointments.video_room_url`
- `appointments.consultation_notes`
- `appointments.patient_joined_at`
- `appointments.doctor_joined_at`

---

## Use Case Relationships

### Include Relationships

```
UC1 (Register) ──includes──> UC7 (Verify Email)
                 includes──> UC28 (Validate Credentials)

UC2 (Login) ──includes──> UC28 (Validate Credentials)

UC11 (Book) ──includes──> UC17 (Check Availability)
             includes──> UC27 (Send Notifications)

UC13 (Cancel) ──includes──> UC27 (Send Notifications)

UC18 (Join Video) ──includes──> UC22 (Generate Video Room)
UC19 (Start Video) ──includes──> UC22 (Generate Video Room)
```

### Extend Relationships (Stretch Goals)

```
UC11 (Book) ──extends──> UC32 (Make Payment) [if payment required - Week 7]

UC20 (Add Notes) ──extends──> UC29 (Create Prescription) [if prescribed - Week 7]
```

---

## Implementation Roadmap

### Week 1-2: Foundation ✅
**Deliverables:**
- All UML diagrams (Use Case, ERD, DFD, Sequence, Class)
- Database schema design (7 core tables + 5 optional)
- Authentication system (JWT, bcrypt, email verification)
- Basic user management API

**Use Cases:** UC1-UC7, UC28 (8 use cases)

---

### Week 3-4: Core Function 1 - Appointment System 🎯
**Deliverables:**
- Complete appointment booking workflow
- Doctor availability management
- Real-time availability checking
- Email notifications
- React UI for browsing/booking

**Use Cases:** UC8-UC17 (10 use cases)

---

### Week 5: Appointment Polish & Reviews ✨
**Deliverables:**
- Appointment cancellation
- Doctor/patient dashboards
- Rating and review system
- End-to-end testing

**Use Cases:** UC24-UC26 (3 use cases)

---

### Week 6: Core Function 2 - Video Consultation 🎥
**Deliverables:**
- Jitsi Meet integration
- Video room generation
- Consultation notes interface
- Patient details view
- Video quality testing

**Use Cases:** UC18-UC23 (6 use cases)

---

### Week 7: Stretch Goals - Prescriptions/Payments 🏆
**Deliverables (if time permits):**
- Basic prescription creation form
- Simple prescription list view
- Stripe Checkout integration
- Medical history UI

**Use Cases:** UC29-UC35 (7 use cases)

---

### Week 8: Testing & Deployment 🚀
**Deliverables:**
- Comprehensive testing (unit, integration, E2E)
- Bug fixes and security audit
- UI/UX polish
- Documentation
- Cloud deployment (Heroku/AWS)
- Demo video

---

## Testing Scenarios

### Scenario 1: Complete Patient Journey (MVP)

1. Patient registers account → UC1
2. Verifies email → UC7
3. Logs in → UC2
4. Browses doctors → UC8
5. Searches by specialization → UC9
6. Views doctor profile → UC10
7. Books appointment → UC11
8. Receives confirmation email → UC27
9. Joins video consultation → UC18
10. Rates doctor → UC24
11. Logs out → UC3

**Expected Duration:** 15-20 minutes
**Success Rate Target:** 100%

---

### Scenario 2: Complete Doctor Journey (MVP)

1. Doctor registers account → UC1
2. Verifies email → UC7
3. Logs in → UC2
4. Sets availability → UC14
5. Views appointment request → UC15
6. Accepts appointment → UC16
7. Starts video consultation → UC19
8. Adds consultation notes → UC20
9. Ends consultation → UC23
10. Views reviews → UC26
11. Logs out → UC3

**Expected Duration:** 15-20 minutes
**Success Rate Target:** 100%

---

## Success Metrics

### Must-Have (Weeks 1-6)

- ✅ **Authentication**: Registration, login, email verification working
- ✅ **Appointment System**: Complete booking workflow functional
- ✅ **Video Consultation**: Successful video calls between parties
- ✅ **Notifications**: Email confirmations for all key events
- ✅ **Reviews**: Basic rating system functional

### Nice-to-Have (Weeks 7-8)

- ⭐ **Prescriptions**: Simple prescription creation and viewing
- ⭐ **Payments**: Stripe Checkout integration
- ⭐ **Medical History**: Patient can add/view conditions
- ⭐ **Admin Dashboard**: Basic user management

---

## Technical Notes

### Simplified Integrations

**Video:** Use Jitsi Meet iframe embed (simplest approach)
```html
<iframe allow="camera; microphone"
        src="https://meet.jit.si/appointment_12345">
</iframe>
```

**Email:** SendGrid with simple templates
```javascript
sgMail.send({
  to: patient.email,
  subject: 'Appointment Confirmed',
  text: `Your appointment with Dr. ${doctor.name} is confirmed`
});
```

**Payment (Optional):** Stripe Checkout (hosted page - easiest)
```javascript
const session = await stripe.checkout.sessions.create({
  payment_method_types: ['card'],
  line_items: [{
    price_data: {
      currency: 'usd',
      product_data: { name: 'Consultation Fee' },
      unit_amount: consultationFee * 100
    },
    quantity: 1
  }],
  mode: 'payment',
  success_url: `${domain}/appointments/{APPOINTMENT_ID}`,
  cancel_url: `${domain}/appointments`
});
```

---

## Database Schema Summary

### Core Tables (7)
1. `users` - Authentication
2. `patients` - Patient profiles
3. `doctors` - Doctor profiles
4. `appointments` - Bookings + video + payment (consolidated)
5. `doctor_availability` - Weekly schedules
6. `reviews` - Ratings and feedback
7. `notifications` - Email tracking

### Optional Tables (5 - Week 7-8)
8. `prescriptions` - Prescription headers
9. `prescription_items` - Medications
10. `medications` - Drug database
11. `medical_history` - Patient conditions
12. `admins` - Admin accounts

---

**Document Version:** 2.0 (Revised for MVP)
**Project Duration:** 8 Weeks
**Last Updated:** October 2025
**Core Use Cases:** 28 (Must-Have) + 8 (Stretch Goals)
