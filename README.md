# TravelMemory – AWS Deployment

This is my deployment of the TravelMemory MERN stack project on AWS.

I have deployed the frontend and backend on EC2 instances and configured Nginx, PM2, and Application Load Balancer to handle traffic. The application is also accessible over HTTPS using CloudFront.

The main goal was to understand how a full-stack app works in a real deployment setup with multiple instances and load balancing.

---

## Live Application

https://d20ahthcxav8kv.cloudfront.net

---

## Architecture

User → CloudFront → ALB → EC2 instances → MongoDB Atlas

- Frontend: React app served using Nginx (2 instances)
- Backend: Node.js API running with PM2 (2 instances)
- Database: MongoDB Atlas
- Load balancing: AWS ALB
- HTTPS: CloudFront

---

## What I implemented

- Deployed full MERN stack on AWS EC2
- Configured Nginx for both frontend and backend
- Used PM2 to keep backend running
- Created multiple frontend and backend instances
- Set up Application Load Balancer with routing rules
- Integrated MongoDB Atlas database
- Enabled HTTPS using CloudFront

---

## Issues I faced

- Mixed content error (HTTP vs HTTPS)
- CORS issue between frontend and backend
- Blank screen due to wrong frontend build on multiple servers
- CloudFront caching problems

Fixed them by:
- using `/api` instead of full backend URL
- rebuilding frontend properly on both instances
- fixing nginx static config
- adjusting CloudFront settings

---
