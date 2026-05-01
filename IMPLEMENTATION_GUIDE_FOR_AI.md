# Implementation Guide for AI Developer

## Architecture Overview
- **Client-Server Architecture**: This application consists of a frontend running in the browser and a backend that handles API requests.
- **Frontend**: Built with React for the user interface, provides interactive elements for prompt building.
- **Backend**: Node.js with Express for API endpoints, supporting data persistence and integration with AI services.
- **Database**: MongoDB for storing user data and prompt configurations.

## Technical Stack
- **Frontend**: 
  - HTML, CSS, JavaScript
  - React
  - Axios (for API requests)
- **Backend**:
  - Node.js
  - Express.js
  - MongoDB
- **Deployment**:
  - Heroku or Vercel for hosting the frontend
  - MongoDB Atlas for database as a service

## Implementation Steps
1. **Set Up Environment**:
   - Install Node.js and npm.
   - Set up a new React app using `create-react-app`.
   - Set up a new Express app for the backend.
2. **Design Database Schema**:
   - Create a MongoDB database.
   - Define schemas for users and prompts.
3. **Build Frontend**:
   - Create main components: Header, PromptBuilder, and PromptsList.
   - Implement routing to navigate between components.
   - Use Axios to connect to the backend API.
4. **Build Backend**:
   - Set up API routes for user authentication, prompt creation, fetching prompts, etc.
   - Implement middleware for handling requests and sending responses.
5. **Integrate AI Services**:
   - Choose an AI service provider (e.g., OpenAI or similar).
   - Write functions to interact with the AI API using prompts.
6. **Testing**:
   - Write unit tests for both frontend and backend functionalities.
   - Conduct integration testing.
   - Perform user acceptance testing.
7. **Deployment**:
   - Deploy the frontend to a hosting service (e.g., Vercel).
   - Deploy the backend to a service such as Heroku and configure the environment variables.

## Quality Checklist
- Code follows best practices and is well-documented.
- Ensure UI is responsive and user-friendly.
- Perform security checks for API and user data handling.
- Validate user inputs for error handling.
- Confirm successful deployment and functionality in the production environment.

## Code Templates
- **React Component Template**:
  ```javascript
  import React from 'react';

  const ComponentName = () => {
      return (
          <div>
              {/* Component code here */}
          </div>
      );
  };

  export default ComponentName;
  ```
- **Express Route Template**:
  ```javascript
  const express = require('express');
  const router = express.Router();

  router.get('/api/path', (req, res) => {
      // API logic here
  });

  module.exports = router;
  ```

---
This implementation guide aims to provide a clear path for AI developers to successfully build the browser-only prompt builder application. Follow these steps carefully to ensure all aspects are covered.