# WhatsApp Agent Workflow

## Overview
The WhatsApp Agent workflow is designed to facilitate communication through WhatsApp using automated processes. This workflow integrates with the WhatsApp API to enable sending and receiving messages, managing contacts, and automating responses.

## Architecture
The architecture of the WhatsApp Agent consists of several components that work together to achieve seamless integration with the WhatsApp platform. It includes:
- **WhatsApp API**: The core service used for communication.
- **n8n Workflow**: A series of nodes that define the logic and process flow.
- **Database**: To store user messages and interaction history.

## Features
- Send and receive messages through WhatsApp.
- Automated responses based on user queries.
- Integration with other services and APIs.
- User management and contact handling.

## Setup Instructions
1. **Clone the Repository**: 
   ```bash
   git clone https://github.com/nxtrohith/abcd-agentic-training-vnr-nxtrohith.git
   cd abcd-agentic-training-vnr-nxtrohith/project1-n8n/WhatsApp Agent
   ```
2. **Install Dependencies**: 
   ```bash
   npm install
   ```
3. **Configure WhatsApp API Credentials**: 
   Update the `.env` file with your WhatsApp API credentials.
4. **Run the Workflow**: 
   ```bash
   n8n start
   ```

## Workflow Nodes Explanation
- **Webhook Node**: Listens for incoming messages from WhatsApp.
- **Function Node**: Processes incoming messages and generates responses.
- **HTTP Request Node**: Sends requests to external APIs if required.

## Usage Examples
- **Sending a Welcome Message**: The workflow can be triggered to send a welcome message when a user initiates a chat.
- **Automated FAQs**: Set up responses to frequently asked questions using the Function Node.

For further customization and advanced features, refer to the official n8n documentation.
