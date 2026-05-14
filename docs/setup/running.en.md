# Running Instructions

To run the **Watheq** system in your local development environment, follow these steps using your Terminal.

## 1. Start Database
Go to the project root directory and start the database container using Docker:
```bash
docker-compose up -d pgadmin
```

## 2. Start Backend Server
Navigate to the `backend` folder, install dependencies, and start:
```bash
cd backend
npm install
npx prisma generate
npx prisma db push
npm run dev
```

## 3. Start Frontend Client
Open a new Terminal window, navigate to the `web` folder:
```bash
cd web
npm install
npm run dev
```

You will get an access link (usually `http://localhost:5173`).

![System Screenshot](assets/placeholder.png)
*(This image will be replaced with a screenshot of the login interface later)*
