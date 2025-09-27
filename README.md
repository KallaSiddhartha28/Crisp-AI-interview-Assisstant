# Crisp-AI-interview-Assisstant
Crisp is an AI-powered interview simulation and management platform designed to streamline the technical screening process for Full Stack (React/Node) roles. This application provides a real-time, timed interview experience for candidates and a comprehensive dashboard for interviewers to review scores, summaries, and chat history.
🤖 Crisp AI Interview Assistant
Overview
Crisp is an AI-powered interview simulation and management platform designed to streamline the technical screening process for Full Stack (React/Node) roles. This application provides a real-time, timed interview experience for candidates and a comprehensive dashboard for interviewers to review scores, summaries, and chat history.

This project was developed as an assignment to demonstrate skills in React development, state management, persistent local storage, and integration with AI APIs (e.g., Gemini API) for dynamic content generation and evaluation.

🚀 Core Features
The application is split into two distinct, synchronized interfaces: the Interviewee (Chat) Tab and the Interviewer (Dashboard) Tab.

1. Interviewee (Chat) Tab
Resume Parsing & Profile Generation:

Allows candidates to upload a PDF or DOCX resume.

Uses AI to extract essential fields: Name, Email, and Phone Number.

If any key field is missing, a chatbot proactively prompts the candidate to fill in the gaps before the interview starts.

Timed AI Interview Flow:

The AI dynamically generates 6 technical questions tailored for a Full Stack (React/Node) role.

The flow consists of: 2 Easy → 2 Medium → 2 Hard questions.

Each question is displayed one at a time with a strict timer:

Easy: 20 seconds

Medium: 60 seconds

Hard: 120 seconds

Answers are automatically submitted when the timer expires, moving the candidate to the next question.

Final Assessment:

After the 6th question, the AI calculates a final score and generates a concise summary of the candidate's performance.

2. Interviewer (Dashboard) Tab
Candidate Management:

Displays a list of all candidates, primarily ordered by their final AI score.

Includes Search and Sort functionality for easy filtering.

Detailed Review:

Clicking on a candidate opens a detailed view.

Shows the complete candidate profile (Name, Email, Phone).

Displays the full chat history of the interview.

Provides question-by-question scoring and the overall final AI summary.

3. Data Persistence & State Management
Resilience: All interview state (timers, submitted answers, progress) is persisted locally.

Restoration: If a candidate closes the browser or refreshes the page, their session is automatically restored upon reopening.

UX: A "Welcome Back" modal is shown to candidates who return to an unfinished session.

🛠️ Technology Stack
Category

Technology

Purpose

Frontend

React

Core library for building the user interface.

State Management

Redux Toolkit

Centralized, predictable state management.

Persistence

Redux-Persist / IndexedDB

Local, resilient data storage for session restoration.

Styling/UI

Tailwind CSS (or similar)

Utility-first CSS framework for a responsive, modern, and clean design.

AI Integration

Gemini API

Used for dynamic question generation, answer evaluation, and final summary creation.

Development

Vite / Next.js (if applicable)

Modern build tooling/framework.

🚦 Installation and Local Setup
Prerequisites
Node.js (v18+)

npm or yarn

Steps
Clone the repository:

git clone [https://github.com/YourUsername/crisp-ai-interview-assistant.git](https://github.com/YourUsername/crisp-ai-interview-assistant.git)
cd crisp-ai-interview-assistant

Install dependencies:

npm install
# or
yarn install

Set up Environment Variables:
Create a file named .env in the root directory and add your Gemini API Key:

VITE_GEMINI_API_KEY="YOUR_API_KEY_HERE"

Run the application:

npm run dev
# or
yarn dev

The application will typically start on http://localhost:5173 (or similar port).

👤 Author
Kalla Siddhartha

