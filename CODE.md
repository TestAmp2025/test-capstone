# Day 28: Capstone Automation Framework

## Complete Implementation Guide

This document contains all the code files needed to implement the K12 Harmony Hub automation framework from scratch.

---

## Table of Contents

1. [Project Setup](#project-setup)
2. [Utilities](#utilities)
3. [Page Objects](#page-objects)
4. [Test Data](#test-data)
5. [Test Specifications](#test-specifications)
6. [Configuration](#configuration)
7. [CI/CD](#cicd)
8. [Running the Tests](#running-the-tests)

---

## Project Setup

### Prerequisites

Ensure you have the following installed on your system before proceeding with the setup.

- **Node.js** (v18 or higher)
- **npm** (comes with Node.js)
- **Git** (for version control)

### Installation Steps

Follow these steps to set up the project from scratch.

1. **Create the project directory structure**:

```bash
mkdir -p k12-automation/{pages,tests,test-data,utils,.github/workflows}
cd k12-automation
```

2. **Initialize npm**:

```bash
npm init -y
```

3. **Install dependencies**:

```bash
npm install -D @playwright/test@^1.40.0 @types/node@^20.10.0 allure-playwright@^2.15.1
```

4. **Install Playwright browsers**:

```bash
npx playwright install --with-deps
```

5. **Install  Allure**:

```bash
npm install -D allure-commandline
```

---

## Utilities

### File: `utils/DateUtils.js`

This utility provides dynamic date generation functions critical for schedule-based testing.

```javascript
/**
 * DateUtils.js
 * Utility functions for handling dynamic date generation and formatting
 * Critical for tests that rely on current date/time (e.g., scheduling features)
 */

class DateUtils {
  /**
   * Get the current date formatted as "DayOfWeek, Month Day, Year"
   * Example: "Tuesday, December 30, 2025"
   * This matches the format used by the K12 Harmony Hub application
   * @returns {string} Formatted date string
   */
  static getFormattedDate() {
    const today = new Date();
    
    const options = {
      weekday: 'long',
      year: 'numeric',
      month: 'long',
      day: 'numeric'
    };
    
    return today.toLocaleDateString('en-US', options);
  }

  /**
   * Get the current day number (1-31)
   * Useful for selecting a specific day in a calendar grid
   * @returns {number} Day of the month
   */
  static getCurrentDay() {
    return new Date().getDate();
  }

  /**
   * Get the current month name (e.g., "December")
   * @returns {string} Full month name
   */
  static getCurrentMonth() {
    const options = { month: 'long' };
    return new Date().toLocaleDateString('en-US', options);
  }

  /**
   * Get the current year (e.g., 2025)
   * @returns {number} Current year
   */
  static getCurrentYear() {
    return new Date().getFullYear();
  }

  /**
   * Get a time string in HH:MM format
   * @param {number} hours - Hour (0-23)
   * @param {number} minutes - Minutes (0-59)
   * @returns {string} Time in HH:MM format
   */
  static formatTime(hours, minutes) {
    const h = hours.toString().padStart(2, '0');
    const m = minutes.toString().padStart(2, '0');
    return `${h}:${m}`;
  }

  /**
   * Get current time in HH:MM format
   * @returns {string} Current time
   */
  static getCurrentTime() {
    const now = new Date();
    return this.formatTime(now.getHours(), now.getMinutes());
  }
}

export { DateUtils };
```

**Key Features:**
- Generates dates in the exact format the application expects
- Prevents hard-coded dates that cause tests to fail over time
- Provides reusable date manipulation methods

---

## Page Objects

### File: `pages/StudentPage.js`

The StudentPage Page Object encapsulates all interactions with the Student Management page.

```javascript
import { expect } from '@playwright/test';

/**
 * StudentPage.js
 * Page Object Model for the Student Management page
 * Handles all interactions related to viewing, adding, and verifying students
 */

class StudentPage {
  /**
   * @param {import('@playwright/test').Page} page
   */
  constructor(page) {
    this.page = page;
    
    // Locators
    this.pageTitle = page.locator('h1', { hasText: 'Student Management' });
    this.pageSubtitle = page.locator('text=Manage your students and their information');
    this.addStudentButton = page.locator('button', { hasText: 'Add Student' });
    this.searchInput = page.locator('input[placeholder="Search students..."]');
    this.studentCountBadge = page.locator('text=/\\d+ Students/');
    
    // Modal locators
    this.modal = page.locator('div[role="dialog"]');
    this.modalTitle = this.modal.locator('h2', { hasText: 'Add New Student' });
    this.inputStudentName = this.modal.locator('input[placeholder*="full name"]');
    this.inputStudentEmail = this.modal.locator('input[placeholder*="email"]');
    this.inputPhone = this.modal.locator('input[placeholder*="123-4567"]');
    this.inputAddress = this.modal.locator('input[placeholder*="address"]');
    this.selectGrade = this.modal.locator('button:has-text("Select grade")');
    this.inputParentName = this.modal.locator('input[placeholder*="Parent"]');
    this.inputParentEmail = this.modal.locator('input[placeholder*="parent@email"]');
    this.submitButton = this.modal.locator('button', { hasText: 'Add Student' });
    this.cancelButton = this.modal.locator('button', { hasText: 'Cancel' });
    
    // Student card locators
    this.studentCards = page.locator('div:has(button:has-text("Edit"))').filter({ has: page.locator('text=/Grade \\d+/') });
  }

  /**
   * Navigate to the Student Management page
   */
  async navigate() {
    await this.page.goto('https://k12-harmony-hub.lovable.app/students');
    await this.pageTitle.waitFor({ state: 'visible' });
  }

  /**
   * Get the page title text
   * @returns {Promise<string>}
   */
  async getPageTitle() {
    return await this.pageTitle.textContent();
  }

  /**
   * Verify the page title is correct
   */
  async verifyPageTitle() {
    await expect(this.pageTitle).toHaveText('Student Management');
  }

  /**
   * Get a student card by name
   * @param {string} studentName - The name of the student
   * @returns {import('@playwright/test').Locator}
   */
  getStudentCardByName(studentName) {
    return this.page.locator(`div:has-text("${studentName}")`).filter({ has: this.page.locator('text=/Grade \\d+/') }).first();
  }

  /**
   * Verify a student is present on the page
   * @param {string} studentName - The name of the student to verify
   */
  async verifyStudentPresent(studentName) {
    const studentCard = this.getStudentCardByName(studentName);
    await expect(studentCard).toBeVisible();
  }

  /**
   * Get the total number of students displayed
   * @returns {Promise<number>}
   */
  async getStudentCount() {
    const badgeText = await this.studentCountBadge.textContent();
    const match = badgeText.match(/(\d+)/);
    return match ? parseInt(match[1]) : 0;
  }

  /**
   * Click the Add Student button to open the modal
   */
  async clickAddStudent() {
    await this.addStudentButton.click();
    await this.modal.waitFor({ state: 'visible' });
  }

  /**
   * Select a grade from the dropdown
   * @param {string} grade - Grade to select (e.g., "Grade 10")
   */
  async selectGradeOption(grade) {
    await this.selectGrade.click();
    await this.page.locator(`div[role="option"]:has-text("${grade}")`).click();
  }

  /**
   * Add a new student with complete details
   * @param {Object} studentData - Student information
   * @param {string} studentData.name - Full name
   * @param {string} studentData.email - Student email
   * @param {string} studentData.grade - Grade level (e.g., "Grade 10")
   * @param {string} [studentData.phone] - Phone number (optional)
   * @param {string} [studentData.address] - Address (optional)
   * @param {string} [studentData.parentName] - Parent/Guardian name (optional)
   * @param {string} [studentData.parentEmail] - Parent email (optional)
   */
  async addStudent(studentData) {
    // Open modal
    await this.clickAddStudent();
    
    // Fill required fields
    await this.inputStudentName.fill(studentData.name);
    await this.inputStudentEmail.fill(studentData.email);
    await this.selectGradeOption(studentData.grade);
    
    // Fill optional fields if provided
    if (studentData.phone) {
      await this.inputPhone.fill(studentData.phone);
    }
    
    if (studentData.address) {
      await this.inputAddress.fill(studentData.address);
    }
    
    if (studentData.parentName) {
      await this.inputParentName.fill(studentData.parentName);
    }
    
    if (studentData.parentEmail) {
      await this.inputParentEmail.fill(studentData.parentEmail);
    }
    
    // Submit
    await this.submitButton.click();
    
    // Wait for modal to close
    await this.modal.waitFor({ state: 'hidden' });
  }

  /**
   * Search for a student by name
   * @param {string} searchTerm - Search term
   */
  async searchStudent(searchTerm) {
    await this.searchInput.fill(searchTerm);
    await this.page.waitForTimeout(500); // Wait for search to filter
  }

  /**
   * Get all visible student names
   * @returns {Promise<string[]>}
   */
  async getAllStudentNames() {
    const cards = await this.studentCards.all();
    const names = [];
    
    for (const card of cards) {
      const nameElement = card.locator('text=/^[A-Z][a-z]+ [A-Z][a-z]+$/').first();
      const name = await nameElement.textContent();
      names.push(name.trim());
    }
    
    return names;
  }
}

export { StudentPage };
```

**Key Features:**
- Abstracts all DOM interactions
- Provides clean, readable methods for tests
- Handles modal interactions automatically
- Supports optional fields

---

### File: `pages/SchedulePage.js`

The SchedulePage Page Object handles calendar interactions and event management.

```javascript
import { expect } from '@playwright/test';
import { DateUtils } from '../utils/DateUtils.js';

/**
 * SchedulePage.js
 * Page Object Model for the Class Schedule page
 * Handles calendar interactions, event creation, and dynamic date handling
 */

class SchedulePage {
  /**
   * @param {import('@playwright/test').Page} page
   */
  constructor(page) {
    this.page = page;
    
    // Page locators
    this.pageTitle = page.locator('h1', { hasText: 'Class Schedule' });
    this.pageSubtitle = page.locator('text=Manage your daily schedule and events');
    this.addEventButton = page.locator('button', { hasText: 'Add Event' });
    
    // Calendar locators
    this.calendarSection = page.locator('text=Calendar').locator('..');
    this.monthYearDisplay = page.locator('text=/^[A-Z][a-z]+ \\d{4}$/');
    this.prevMonthButton = page.locator('button[hint="Go to previous month"]');
    this.nextMonthButton = page.locator('button[hint="Go to next month"]');
    this.calendarGrid = page.locator('button[role="gridcell"]');
    
    // Schedule display locators
    this.scheduleHeader = page.locator('text=/^Schedule for/');
    this.emptyStateMessage = page.locator('text=No events scheduled');
    this.emptyStateSubtext = page.locator('text=Add an event to get started.');
    
    // Modal locators
    this.modal = page.locator('div[role="dialog"]');
    this.modalTitle = this.modal.locator('h2', { hasText: 'Schedule New Event' });
    this.inputEventTitle = this.modal.locator('input#title');
    this.selectType = this.modal.locator('button[role="combobox"]').filter({ hasText: 'Class' });
    this.selectSubject = this.modal.locator('button[role="combobox"]').filter({ hasText: 'Select subject' });
    this.selectStartTime = this.modal.locator('button[role="combobox"]').filter({ hasText: 'Select start time' });
    this.selectEndTime = this.modal.locator('button[role="combobox"]').filter({ hasText: 'Select end time' });
    this.inputLocation = this.modal.locator('input#location');
    this.inputAttendees = this.modal.locator('input#attendees');
    this.textareaDescription = this.modal.locator('textarea#description');
    this.scheduleEventButton = this.modal.locator('button', { hasText: 'Schedule Event' });
    this.cancelButton = this.modal.locator('button', { hasText: 'Cancel' });
    this.closeButton = this.modal.locator('button', { hasText: 'Close' });
  }

  /**
   * Navigate to the Schedule page
   */
  async navigate() {
    await this.page.goto('https://k12-harmony-hub.lovable.app/schedule');
    await this.pageTitle.waitFor({ state: 'visible' });
  }

  /**
   * Get the page title text
   * @returns {Promise<string>}
   */
  async getPageTitle() {
    return await this.pageTitle.textContent();
  }

  /**
   * Verify the page title is correct
   */
  async verifyPageTitle() {
    await expect(this.pageTitle).toHaveText('Class Schedule');
  }

  /**
   * Get the schedule header text (e.g., "Schedule for Tuesday, December 30, 2025")
   * @returns {Promise<string>}
   */
  async getScheduleHeaderText() {
    return await this.scheduleHeader.textContent();
  }

  /**
   * Verify today's date is displayed correctly in the schedule header
   */
  async verifyTodaysDate() {
    const expectedDate = DateUtils.getFormattedDate();
    const headerText = await this.getScheduleHeaderText();
    
    // Header format: "Schedule for Tuesday, December 30, 2025"
    // Expected format from DateUtils: "Tuesday, December 30, 2025"
    expect(headerText).toContain(expectedDate);
  }

  /**
   * Get a calendar day button by day number
   * @param {number} dayNumber - Day of the month (1-31)
   * @returns {import('@playwright/test').Locator}
   */
  getDayButton(dayNumber) {
    return this.page.locator(`button[role="gridcell"]:has-text("${dayNumber}")`).first();
  }

  /**
   * Get today's calendar day button dynamically
   * @returns {import('@playwright/test').Locator}
   */
  getTodayButton() {
    const today = DateUtils.getCurrentDay();
    return this.getDayButton(today);
  }

  /**
   * Click on a specific day in the calendar
   * @param {number} dayNumber - Day of the month
   */
  async clickDay(dayNumber) {
    await this.getDayButton(dayNumber).click();
  }

  /**
   * Click on today's date in the calendar
   */
  async clickToday() {
    const today = DateUtils.getCurrentDay();
    await this.clickDay(today);
  }

  /**
   * Click the Add Event button to open the modal
   */
  async clickAddEvent() {
    await this.addEventButton.click();
    await this.modal.waitFor({ state: 'visible' });
  }

  /**
   * Select an option from a dropdown
   * @param {import('@playwright/test').Locator} dropdownLocator - The dropdown button
   * @param {string} optionText - The option to select
   */
  async selectDropdownOption(dropdownLocator, optionText) {
    await dropdownLocator.click();
    await this.page.locator(`div[role="option"]:has-text("${optionText}")`).click();
  }

  /**
   * Add a new event with complete details
   * @param {Object} eventData - Event information
   * @param {string} eventData.title - Event title
   * @param {string} [eventData.type] - Event type (e.g., "Class", "Meeting")
   * @param {string} [eventData.subject] - Subject (e.g., "Mathematics")
   * @param {string} [eventData.startTime] - Start time (e.g., "09:00")
   * @param {string} [eventData.endTime] - End time (e.g., "10:00")
   * @param {string} [eventData.location] - Location/room
   * @param {number} [eventData.attendees] - Expected number of attendees
   * @param {string} [eventData.description] - Event description
   */
  async addEvent(eventData) {
    // Open modal
    await this.clickAddEvent();
    
    // Fill required field
    await this.inputEventTitle.fill(eventData.title);
    
    // Fill optional fields if provided
    if (eventData.type) {
      await this.selectDropdownOption(this.selectType, eventData.type);
    }
    
    if (eventData.subject) {
      await this.selectDropdownOption(this.selectSubject, eventData.subject);
    }
    
    if (eventData.startTime) {
      await this.selectDropdownOption(this.selectStartTime, eventData.startTime);
    }
    
    if (eventData.endTime) {
      await this.selectDropdownOption(this.selectEndTime, eventData.endTime);
    }
    
    if (eventData.location) {
      await this.inputLocation.fill(eventData.location);
    }
    
    if (eventData.attendees !== undefined) {
      await this.inputAttendees.fill(eventData.attendees.toString());
    }
    
    if (eventData.description) {
      await this.textareaDescription.fill(eventData.description);
    }
    
    // Submit
    await this.scheduleEventButton.click();
    
    // Wait for modal to close
    await this.modal.waitFor({ state: 'hidden' });
  }

  /**
   * Get an event card by title
   * @param {string} eventTitle - The title of the event
   * @returns {import('@playwright/test').Locator}
   */
  getEventCardByTitle(eventTitle) {
    return this.page.locator(`div:has-text("${eventTitle}")`).filter({ 
      has: this.page.locator('button:has-text("Edit"), button:has-text("Delete")') 
    }).first();
  }

  /**
   * Verify an event is present in the schedule
   * @param {string} eventTitle - The title of the event to verify
   */
  async verifyEventPresent(eventTitle) {
    const eventCard = this.getEventCardByTitle(eventTitle);
    await expect(eventCard).toBeVisible();
  }

  /**
   * Click the Edit button for a specific event
   * @param {string} eventTitle - The title of the event to edit
   */
  async clickEditEvent(eventTitle) {
    const eventCard = this.getEventCardByTitle(eventTitle);
    await eventCard.locator('button:has-text("Edit")').click();
    await this.modal.waitFor({ state: 'visible' });
  }

  /**
   * Edit an existing event
   * @param {string} eventTitle - The title of the event to edit
   * @param {Object} newData - New event data (same structure as addEvent)
   */
  async editEvent(eventTitle, newData) {
    // Open edit modal
    await this.clickEditEvent(eventTitle);
    
    // Update fields if provided
    if (newData.title) {
      await this.inputEventTitle.clear();
      await this.inputEventTitle.fill(newData.title);
    }
    
    if (newData.type) {
      await this.selectDropdownOption(this.selectType, newData.type);
    }
    
    if (newData.subject) {
      await this.selectDropdownOption(this.selectSubject, newData.subject);
    }
    
    if (newData.startTime) {
      await this.selectDropdownOption(this.selectStartTime, newData.startTime);
    }
    
    if (newData.endTime) {
      await this.selectDropdownOption(this.selectEndTime, newData.endTime);
    }
    
    if (newData.location) {
      await this.inputLocation.clear();
      await this.inputLocation.fill(newData.location);
    }
    
    if (newData.attendees !== undefined) {
      await this.inputAttendees.clear();
      await this.inputAttendees.fill(newData.attendees.toString());
    }
    
    if (newData.description) {
      await this.textareaDescription.clear();
      await this.textareaDescription.fill(newData.description);
    }
    
    // Save changes (button text might be "Save" or "Update Event")
    await this.page.locator('button:has-text("Save"), button:has-text("Update")').click();
    
    // Wait for modal to close
    await this.modal.waitFor({ state: 'hidden' });
  }

  /**
   * Verify the empty state is displayed
   */
  async verifyEmptyState() {
    await expect(this.emptyStateMessage).toBeVisible();
    await expect(this.emptyStateSubtext).toBeVisible();
  }

  /**
   * Get the current month and year from the calendar
   * @returns {Promise<string>}
   */
  async getMonthYear() {
    return await this.monthYearDisplay.textContent();
  }
}

export { SchedulePage };
```

**Key Features:**
- Integrates DateUtils for dynamic date handling
- Handles complex calendar interactions
- Supports event creation and editing
- Provides flexible dropdown selection

---

## Test Data

### File: `test-data/student.data.js`

Centralized test data for student management tests.

```javascript
/**
 * student.data.js
 * Test data for Student Management tests
 * Contains sample student records for data-driven testing
 */

const studentData = {
  // Existing students (already in the system)
  existingStudents: [
    {
      name: 'Emma Johnson',
      grade: 'Grade 10',
      email: 'emma.j@email.com',
      phone: '(555) 123-4567',
      address: '123 Oak Street, Springfield',
      parentName: 'Sarah Johnson',
      enrolledDate: '8/15/2024'
    },
    {
      name: 'Michael Chen',
      grade: 'Grade 11',
      email: 'michael.c@email.com',
      phone: '(555) 234-5678',
      address: '456 Maple Ave, Springfield',
      parentName: 'Lisa Chen',
      enrolledDate: '8/14/2024'
    },
    {
      name: 'Sofia Rodriguez',
      grade: 'Grade 12',
      email: 'sofia.r@email.com',
      phone: '(555) 345-6789',
      address: '789 Pine Road, Springfield',
      parentName: 'Carlos Rodriguez',
      enrolledDate: '8/14/2024'
    }
  ],

  // New students to be added during tests
  newStudents: [
    {
      name: 'Alex Thompson',
      grade: 'Grade 9',
      email: 'alex.thompson@email.com',
      phone: '(555) 111-2222',
      address: '100 Elm Street, Springfield',
      parentName: 'Jennifer Thompson',
      parentEmail: 'jennifer.t@email.com'
    },
    {
      name: 'Maya Patel',
      grade: 'Grade 10',
      email: 'maya.patel@email.com',
      phone: '(555) 222-3333',
      address: '200 Cedar Lane, Springfield',
      parentName: 'Raj Patel',
      parentEmail: 'raj.p@email.com'
    },
    {
      name: 'Jordan Williams',
      grade: 'Grade 11',
      email: 'jordan.w@email.com',
      phone: '(555) 333-4444',
      address: '300 Birch Avenue, Springfield',
      parentName: 'Michelle Williams',
      parentEmail: 'michelle.w@email.com'
    },
    {
      name: 'Zara Ahmed',
      grade: 'Grade 12',
      email: 'zara.ahmed@email.com',
      phone: '(555) 444-5555',
      address: '400 Spruce Drive, Springfield',
      parentName: 'Hassan Ahmed',
      parentEmail: 'hassan.a@email.com'
    }
  ],

  // Invalid data for negative testing (optional)
  invalidData: [
    {
      name: '',
      grade: 'Grade 10',
      email: 'invalid@email.com',
      expectedError: 'Name is required'
    },
    {
      name: 'Test Student',
      grade: 'Grade 10',
      email: 'not-an-email',
      expectedError: 'Invalid email format'
    }
  ]
};

export { studentData };
```

---

### File: `test-data/schedule.data.js`

Centralized test data for schedule management tests.

```javascript
/**
 * schedule.data.js
 * Test data for Schedule Management tests
 * Contains sample event records for data-driven testing
 */

const scheduleData = {
  // Events to be created during tests
  newEvents: [
    {
      title: 'Advanced Mathematics',
      type: 'Class',
      subject: 'Mathematics',
      startTime: '09:00',
      endTime: '10:00',
      location: 'Room B-204',
      attendees: 32,
      description: 'Grade 11 & 12 Advanced Mathematics lecture covering calculus fundamentals.'
    },
    {
      title: 'Chemistry Lab Session',
      type: 'Lab',
      subject: 'Science',
      startTime: '10:30',
      endTime: '12:00',
      location: 'Science Lab 1',
      attendees: 24,
      description: 'Hands-on chemistry experiments for Grade 10 students.'
    },
    {
      title: 'Parent-Teacher Conference',
      type: 'Meeting',
      subject: 'General',
      startTime: '14:00',
      endTime: '15:30',
      location: 'Conference Room A',
      attendees: 15,
      description: 'Quarterly parent-teacher meetings to discuss student progress.'
    },
    {
      title: 'Faculty Meeting',
      type: 'Meeting',
      subject: 'Administration',
      startTime: '15:30',
      endTime: '16:30',
      location: 'Main Conference Room',
      attendees: 25,
      description: 'Monthly faculty meeting to discuss curriculum updates and policies.'
    }
  ],

  // Event updates for edit tests
  eventUpdates: [
    {
      originalTitle: 'Advanced Mathematics',
      newData: {
        title: 'Advanced Mathematics - Review Session',
        location: 'Room B-205',
        attendees: 35,
        description: 'Extended review session covering all topics from this semester.'
      }
    },
    {
      originalTitle: 'Chemistry Lab Session',
      newData: {
        startTime: '11:00',
        endTime: '12:30',
        location: 'Science Lab 2',
        description: 'Rescheduled chemistry lab with updated experiments.'
      }
    }
  ],

  // Event types available in the system
  eventTypes: [
    'Class',
    'Lab',
    'Meeting',
    'Event',
    'Exam',
    'Activity'
  ],

  // Subjects available in the system
  subjects: [
    'Mathematics',
    'Science',
    'English',
    'History',
    'Physical Education',
    'Art',
    'Music',
    'General',
    'Administration'
  ],

  // Time slots (in 30-minute intervals)
  timeSlots: [
    '08:00', '08:30', '09:00', '09:30', '10:00', '10:30',
    '11:00', '11:30', '12:00', '12:30', '13:00', '13:30',
    '14:00', '14:30', '15:00', '15:30', '16:00', '16:30',
    '17:00'
  ]
};

export { scheduleData };
```

---

## Test Specifications

### File: `tests/student.spec.js`

Complete test suite for student management functionality.

```javascript
import { test, expect } from '@playwright/test';
import { StudentPage } from '../pages/StudentPage.js';
import { studentData } from '../test-data/student.data.js';

/**
 * Student Management Test Suite
 * Tests for verifying student management functionality
 */

test.describe('Student Management', () => {
  let studentPage;

  test.beforeEach(async ({ page }) => {
    studentPage = new StudentPage(page);
    await studentPage.navigate();
  });

  /**
   * TC-01: Verify Title on the Page
   * Validates that the Student Management page displays the correct title
   */
  test('TC-01: Verify page title is displayed correctly', async () => {
    // Verify the page title
    await studentPage.verifyPageTitle();
    
    // Additional assertion for subtitle
    await expect(studentPage.pageSubtitle).toBeVisible();
  });

  /**
   * TC-02: Verify Student Name is Present
   * Validates that existing students are visible on the page
   */
  test('TC-02: Verify existing student is present', async () => {
    // Get the first existing student from test data
    const existingStudent = studentData.existingStudents[0];
    
    // Verify the student is present on the page
    await studentPage.verifyStudentPresent(existingStudent.name);
  });

  /**
   * TC-03: Add New Student
   * Tests the ability to add a new student with complete information
   */
  test('TC-03: Add a new student successfully', async () => {
    // Get the first new student from test data
    const newStudent = studentData.newStudents[0];
    
    // Get initial student count
    const initialCount = await studentPage.getStudentCount();
    
    // Add the new student
    await studentPage.addStudent(newStudent);
    
    // Wait for the page to update
    await studentPage.page.waitForTimeout(1000);
    
    // Verify student count increased
    const newCount = await studentPage.getStudentCount();
    expect(newCount).toBe(initialCount + 1);
  });

  /**
   * TC-04: Verify New Student is Added
   * Validates that the newly added student appears in the student list
   */
  test('TC-04: Verify newly added student is visible', async () => {
    // Get a new student from test data
    const newStudent = studentData.newStudents[1]; // Using index 1 for variety
    
    // Add the student
    await studentPage.addStudent(newStudent);
    
    // Wait for the UI to update
    await studentPage.page.waitForTimeout(1000);
    
    // Verify the student is now present
    await studentPage.verifyStudentPresent(newStudent.name);
  });

});

```

---

### File: `tests/schedule.spec.js`

Complete test suite for schedule management functionality with dynamic date handling.

```javascript
import { test, expect } from '@playwright/test';
import { SchedulePage } from '../pages/SchedulePage.js';
import { scheduleData } from '../test-data/schedule.data.js';
import { DateUtils } from '../utils/DateUtils.js';

/**
 * Schedule Management Test Suite
 * Tests for verifying schedule and event management functionality
 * Includes dynamic date handling for real-time testing
 */

test.describe('Schedule Management', () => {
  let schedulePage;

  test.beforeEach(async ({ page }) => {
    schedulePage = new SchedulePage(page);
    await schedulePage.navigate();
  });

  /**
   * TC-SCHED-01: Verify Title on the Page
   * Validates that the Class Schedule page displays the correct title
   */
  test('TC-SCHED-01: Verify page title is displayed correctly', async () => {
    // Verify the page title
    await schedulePage.verifyPageTitle();
    
    // Additional assertion for subtitle
    await expect(schedulePage.pageSubtitle).toBeVisible();
  });

  /**
   * TC-SCHED-02: Verify Today's Date (Dynamic)
   * Validates that the schedule header displays the correct current date
   * This test uses dynamic date generation to ensure it works regardless of when it runs
   */
  test('TC-SCHED-02: Verify today\'s date is displayed correctly', async () => {
    // Click on today's date in the calendar
    await schedulePage.clickToday();
    
    // Verify the schedule header shows today's date
    await schedulePage.verifyTodaysDate();
    
    // Additional verification: Check the format
    const headerText = await schedulePage.getScheduleHeaderText();
    const expectedDate = DateUtils.getFormattedDate();
    
    console.log(`Expected date: ${expectedDate}`);
    console.log(`Actual header: ${headerText}`);
    
    expect(headerText).toContain(expectedDate);
  });

  /**
   * TC-SCHED-03: Create an Event for Today (Dynamic)
   * Tests the ability to create an event for the current date
   * Uses dynamic date logic to ensure the test always runs successfully
   */
  test('TC-SCHED-03: Create an event for today', async () => {
    // Get event data
    const eventData = scheduleData.newEvents[0];
    
    // Click on today's date
    await schedulePage.clickToday();
    
    // Add the event
    await schedulePage.addEvent(eventData);
    
    // Wait for the UI to update
    await schedulePage.page.waitForTimeout(1000);
    
    // Verify the event was created
    await schedulePage.verifyEventPresent(eventData.title);
  });

  /**
   * TC-SCHED-04: Verify New Event is Added
   * Validates that a newly created event appears in the schedule
   */
  test('TC-SCHED-04: Verify newly added event is visible', async () => {
    // Get event data
    const eventData = scheduleData.newEvents[1];
    
    // Click on today's date
    await schedulePage.clickToday();
    
    // Add the event
    await schedulePage.addEvent(eventData);
    
    // Wait for the UI to update
    await schedulePage.page.waitForTimeout(1000);
    
    // Verify the event is visible
    await schedulePage.verifyEventPresent(eventData.title);
    
    // Verify the event card contains key information
    const eventCard = schedulePage.getEventCardByTitle(eventData.title);
    await expect(eventCard).toContainText(eventData.location);
  });

  /**
   * TC-SCHED-05: Edit Event
   * Tests the ability to edit an existing event
   */
  test('TC-SCHED-05: Edit an existing event', async () => {
    // First, create an event
    const originalEvent = scheduleData.newEvents[2];
    await schedulePage.clickToday();
    await schedulePage.addEvent(originalEvent);
    await schedulePage.page.waitForTimeout(1000);
    
    // Get the update data
    const updateData = scheduleData.eventUpdates[0];
    
    // Edit the event
    await schedulePage.editEvent(originalEvent.title, updateData.newData);
    
    // Wait for the UI to update
    await schedulePage.page.waitForTimeout(1000);
    
    // Verify the updated event is visible
    await schedulePage.verifyEventPresent(updateData.newData.title);
  });

  /**
   * TC-SCHED-06: Verify Event is Edited
   * Validates that the edited event displays the updated information
   */
  test('TC-SCHED-06: Verify edited event shows updated information', async () => {
    // Create an event
    const originalEvent = scheduleData.newEvents[3];
    await schedulePage.clickToday();
    await schedulePage.addEvent(originalEvent);
    await schedulePage.page.waitForTimeout(1000);
    
    // Get the update data
    const updateData = scheduleData.eventUpdates[1];
    
    // Edit the event
    await schedulePage.editEvent(originalEvent.title, updateData.newData);
    await schedulePage.page.waitForTimeout(1000);
    
    // Verify the updated location is displayed
    const eventCard = schedulePage.getEventCardByTitle(originalEvent.title);
    await expect(eventCard).toContainText(updateData.newData.location);
  });

  /**
   * BONUS: Verify Empty State
   * Tests that the empty state message is displayed when no events exist
   */
  test('BONUS: Verify empty state when no events scheduled', async () => {
    // Click on a future date that likely has no events
    const futureDay = DateUtils.getCurrentDay() + 5;
    await schedulePage.clickDay(futureDay > 28 ? 15 : futureDay);
    
    // Verify empty state is displayed
    await schedulePage.verifyEmptyState();
  });

  /**
   * BONUS: Data-Driven Test - Create Multiple Events
   * Demonstrates data-driven testing by creating multiple events
   */
  test('BONUS: Create multiple events for today', async () => {
    // Click on today's date
    await schedulePage.clickToday();
    
    // Add multiple events
    for (const eventData of scheduleData.newEvents) {
      await schedulePage.addEvent(eventData);
      await schedulePage.page.waitForTimeout(500);
      
      // Verify each event was added
      await schedulePage.verifyEventPresent(eventData.title);
    }
  });

  /**
   * BONUS: Verify Calendar Navigation
   * Tests the calendar month navigation functionality
   */
  test('BONUS: Navigate calendar months', async () => {
    // Get current month/year
    const currentMonthYear = await schedulePage.getMonthYear();
    console.log(`Current month/year: ${currentMonthYear}`);
    
    // Navigate to next month
    await schedulePage.nextMonthButton.click();
    await schedulePage.page.waitForTimeout(500);
    
    // Verify month changed
    const nextMonthYear = await schedulePage.getMonthYear();
    console.log(`Next month/year: ${nextMonthYear}`);
    expect(nextMonthYear).not.toBe(currentMonthYear);
    
    // Navigate back
    await schedulePage.prevMonthButton.click();
    await schedulePage.page.waitForTimeout(500);
    
    // Verify we're back to the original month
    const backToOriginal = await schedulePage.getMonthYear();
    expect(backToOriginal).toBe(currentMonthYear);
  });
});
```

---

## Configuration

### File: `package.json`

```json
{
  "name": "day-28-k12-harmony-hub-automation",
  "version": "1.0.0",
  "description": "K12 Harmony Hub automation framework with Playwright and Allure",
  "main": "index.js",
  "type": "module",
  "scripts": {
    "test": "npx playwright test",
    "test:debug": "npx playwright test --debug",
    "test:headed": "npx playwright test --headed",
    "test:student": "npx playwright test tests/student.spec.js",
    "test:schedule": "npx playwright test tests/schedule.spec.js",
    "report": "npx playwright show-report",
    "allure:generate": "allure generate ./allure-results --clean -o ./allure-report",
    "allure:open": "allure open ./allure-report",
    "allure:serve": "allure serve ./allure-results"
  },
  "keywords": [
    "playwright",
    "automation",
    "testing",
    "javascript",
    "allure",
    "k12",
    "education"
  ],
  "author": "Testamplify Instructor",
  "license": "ISC",
  "devDependencies": {
    "@playwright/test": "^1.40.0",
    "@types/node": "^20.10.0",
    "allure-playwright": "^2.15.1"
  }
}
```

---

### File: `playwright.config.js`

```javascript
// @ts-check
import { defineConfig, devices } from '@playwright/test';

const isCI = !!process.env.GITHUB_ACTIONS;

export default defineConfig({
  testDir: './tests',

  // 🕒 Global timeouts
  timeout: 30_000,
  expect: { timeout: 5_000 },

  retries: isCI ? 2 : 0,

  // run tests in files in parallel
  fullyParallel: true,

  // let Playwright choose a good number of workers (parallel runs)
  // (you can override on CLI with --workers=6, etc.)
  workers: isCI ? 2 : '100%',

  // 📦 Test output folders
  outputDir: 'test-results',
  
  // 📊 Reporters
  reporter: [
    ['github'],
    ['html', { open: 'never' }],
    ['allure-playwright', { 
      outputFolder: 'allure-results',
      detail: true,
      suiteTitle: true
    }]
  ],

  use: {
    // 💻 Full HD viewport (simulates full-screen desktop)
    viewport: { width: 1920, height: 1080 },

    // ⏱ Action & navigation timeouts
    actionTimeout: 10_000,
    navigationTimeout: 20_000,

    // 🎥 Debug artifacts
    trace: isCI ? 'retain-on-failure' : 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',

    // 🌐 Base URL (optional)
    baseURL: 'https://k12-harmony-hub.lovable.app',
  },

  // 🌍 Multi-browser projects
  projects: [
    { 
      name: 'chromium', 
      use: { ...devices['Desktop Chrome'] } 
    },
    { 
      name: 'firefox',  
      use: { ...devices['Desktop Firefox'] } 
    },
    { 
      name: 'webkit',   
      use: { ...devices['Desktop Safari'] } 
    },
    {
      name: 'edge',
      use: {
        ...devices['Desktop Edge'],
        channel: 'msedge', // runs real Microsoft Edge
      },
    },
  ],
});
```

---

### File: `.gitignore`

```
# Node modules
node_modules/

# Test results
test-results/
playwright-report/
allure-results/
allure-report/

# Environment variables
.env
.env.local

# OS files
.DS_Store
Thumbs.db

# IDE files
.vscode/
.idea/
*.swp
*.swo

# Logs
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# Playwright
playwright/.auth/
```

---

## CI/CD

### File: `.github/workflows/playwright.yml`

Complete GitHub Actions workflow for automated testing and Allure reporting.

```yaml
name: Playwright Tests with Allure Report

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:

jobs:
  test:
    timeout-minutes: 60
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '20'

    - name: Install dependencies
      run: npm ci

    - name: Install Playwright Browsers
      run: npx playwright install --with-deps

    - name: Run Playwright tests
      run: npm test
      continue-on-error: true

    - name: Upload Playwright Report
      uses: actions/upload-artifact@v4
      if: always()
      with:
        name: playwright-report
        path: playwright-report/
        retention-days: 30

    - name: Upload Test Results
      uses: actions/upload-artifact@v4
      if: always()
      with:
        name: test-results
        path: test-results/
        retention-days: 30

    - name: Upload Allure Results
      uses: actions/upload-artifact@v4
      if: always()
      with:
        name: allure-results
        path: allure-results/
        retention-days: 30

  # Generate and deploy Allure Report
  allure-report:
    needs: test
    runs-on: ubuntu-latest
    if: always()
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Download Allure Results
      uses: actions/download-artifact@v4
      with:
        name: allure-results
        path: allure-results

    - name: Setup Java (required for Allure)
      uses: actions/setup-java@v4
      with:
        distribution: 'temurin'
        java-version: '17'

    - name: Install Allure CLI
      run: |
        wget https://github.com/allure-framework/allure2/releases/download/2.24.1/allure-2.24.1.tgz
        tar -zxvf allure-2.24.1.tgz
        sudo mv allure-2.24.1 /opt/allure
        sudo ln -s /opt/allure/bin/allure /usr/bin/allure

    - name: Generate Allure Report
      run: allure generate allure-results --clean -o allure-report

    - name: Upload Allure Report
      uses: actions/upload-artifact@v4
      with:
        name: allure-report
        path: allure-report/
        retention-days: 30

    - name: Deploy to GitHub Pages
      uses: peaceiris/actions-gh-pages@v3
      if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/master'
      with:
        github_token: ${{ secrets.GITHUB_TOKEN }}
        publish_dir: ./allure-report
        publish_branch: gh-pages
        keep_files: false
```

**Important Notes:**
- The workflow runs on push to `main`, `master`, or `develop` branches
- It can also be triggered manually via `workflow_dispatch`
- Allure reports are deployed to GitHub Pages automatically
- To enable GitHub Pages, go to your repository settings and enable it for the `gh-pages` branch

---

## Running the Tests

### Local Execution

Run all tests across all browsers:
```bash
npm test
```

Run tests in headed mode (see the browser):
```bash
npm run test:headed
```

Run tests in debug mode:
```bash
npm run test:debug
```

Run only student tests:
```bash
npm run test:student
```

Run only schedule tests:
```bash
npm run test:schedule
```

View the Playwright HTML report:
```bash
npm run report
```

### Allure Reports

Generate Allure report:
```bash
npm run allure:generate
```

Open the generated Allure report:
```bash
npm run allure:open
```

Generate and serve Allure report in one command:
```bash
npm run allure:serve
```

### CI Execution

The tests run automatically in GitHub Actions when you:
1. Push code to `main`, `master`, or `develop`
2. Create a pull request
3. Manually trigger the workflow from the Actions tab

View the Allure report at:
```
https://[your-username].github.io/[your-repo-name]/
```

---

## Key Takeaways

This framework demonstrates professional automation practices including the following key principles.

**Separation of Concerns**: Page Objects, test data, and test logic are completely separated, making the framework maintainable and scalable.

**Dynamic Data Handling**: The DateUtils utility ensures tests work correctly regardless of when they run, eliminating hard-coded dates that cause failures.

**Data-Driven Testing**: Centralized test data files enable easy expansion of test coverage without modifying test code.

**CI/CD Integration**: GitHub Actions automates test execution and report generation, providing immediate feedback on code changes.

**Professional Reporting**: Allure provides rich, visual test reports with historical trends and detailed execution logs.

**Multi-Browser Support**: Tests run across Chromium, Firefox, WebKit, and Edge to ensure cross-browser compatibility.

This framework serves as a solid foundation for any Playwright automation project and can be adapted to test any web application.

---

**Document Version**: 1.0  
**Last Updated**: December 30, 2025  
**Author**: Testamplify Instructor
