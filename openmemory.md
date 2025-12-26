# Omi Project Index

## Overview
Omi is an AI-powered personal assistant platform focusing on wearable integration and content generation through voice commands. The project consists of a Flutter mobile app, a Python backend, and various firmware/hardware components for the Omi DevKit and Glass.

## Architecture
- **Mobile App**: Flutter based, handles UI/UX and initial processing.
- **Backend**: Python (FastAPI), handles core logic, integrations, and long-running workflows.
- **Integrations**: Webhook-based system connecting Omi to external services like BuildShip, Zapier, etc.
- **Content Generation**: Specialized workflows for generating meditations, music, images, and videos via AI/ML APIs.

## User Defined Namespaces
- backend
- integration
- content-generation
- mobile-app

## Components
- **Memory Webhook**: Endpoint `/omi-memory` (in BuildShip) or `/v1/integrations/workflow/memories` (in backend) for processing conversation transcripts.
- **Workflow Router**: Logic (often in BuildShip) that detects commands like "generate meditation" and routes them to AI models.
- **FFmpeg Processing**: Audio mixing logic for creating polished content (e.g., mixing voice and background music).

## Patterns
- **Memory-First Development**: Standardized workflow for searching and storing codebase understanding.
- **Voice Command Trigger**: Use of "command generate [type] ... command execute" syntax in transcripts.
- **Public URL Delivery**: AI APIs return public CDN URLs, minimizing the need for intermediate storage.


