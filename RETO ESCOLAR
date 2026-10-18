const express = require("express");
const { randomUUID } = require("node:crypto");

const app = express();
const PORT = process.env.PORT || 3000;

app.use(express.json({ limit: "100kb" }));

// Datos de ejemplo para comenzar
const quizzes = [
  {
    id: randomUUID(),
    title: "Multiplicaciones rápidas",
    subject: "Matemáticas",
    questions: [
      {
        question: "¿Cuánto es 8 x 7?",
        options: ["54", "56", "64", "48"],
        correct: 1
      },
      {
        question: "¿Cuánto es 9 x 6?",
        options: ["45", "52", "54", "63"],
        correct: 2
      }
    ]
  },
  {
    id: randomUUID(),
    title: "English Challenge",
    subject: "Inglés",
    questions: [
      {
        question: 'What is the plural of "child"?',
        options: ["Childs", "Children", "Childes", "Childrens"],
        correct: 1
      }
    ]
  }
];

// Perfil de demostración. Más adelante usaremos una base de datos.
const profiles = new Map();

function getProfile(studentId) {
  if (!profiles.has(studentId)) {
    profiles.set(studentId, {
      points: 0,
      completed: 0,
      attempts: []
    });
  }

  return profiles.get(studentId);
}

function getLevel(points) {
  return Math.floor(points / 100) + 1;
}

// Ruta inicial
app.get("/", (req, res) => {
  res.json({
    app: "Reto Escolar",
    message: "¡Bienvenido a la API!",
    endpoints: [
      "GET /api/quizzes",
      "GET /api/quizzes/:id",
      "POST /api/quizzes",
      "POST /api/quizzes/:id/attempts",
      "GET /api/students/:studentId/stats"
    ]
  });
});

// Consultar todos los retos
app.get("/api/quizzes", (req, res) => {
  const result = quizzes.map(quiz => ({
    id: quiz.id,
    title: quiz.title,
    subject: quiz.subject,
    questionCount: quiz.questions.length
  }));

  res.json(result);
});

// Consultar un reto sin revelar las respuestas correctas
app.get("/api/quizzes/:id", (req, res) => {
  const quiz = quizzes.find(q => q.id === req.params.id);

  if (!quiz) {
    return res.status(404).json({
      error: "No se encontró el reto."
    });
  }

  res.json({
    id: quiz.id,
    title: quiz.title,
    subject: quiz.subject,
    questions: quiz.questions.map(q => ({
      question: q.question,
      options: q.options
    }))
  });
});

// Crear un reto (todavía sin autenticación de profesor)
app.post("/api/quizzes", (req, res) => {
  const { title, subject, questions } = req.body;

  if (
    typeof title !== "string" ||
    !title.trim() ||
    title.length > 100 ||
    typeof subject !== "string" ||
    !subject.trim() ||
    subject.length > 60 ||
    !Array.isArray(questions) ||
    questions.length < 1 ||
    questions.length > 50
  ) {
    return res.status(400).json({
      error: "Revisa el título, materia y preguntas."
    });
  }

  const validQuestions = questions.every(q =>
    q &&
    typeof q.question === "string" &&
    q.question.trim().length > 0 &&
    q.question.length <= 500 &&
    Array.isArray(q.options) &&
    q.options.length >= 2 &&
    q.options.length <= 6 &&
    q.options.every(
      option =>
        typeof option === "string" &&
        option.trim().length > 0 &&
        option.length <= 200
    ) &&
    Number.isInteger(q.correct) &&
    q.correct >= 0 &&
    q.correct < q.options.length
  );

  if (!validQuestions) {
    return res.status(400).json({
      error: "Una o más preguntas tienen datos inválidos."
    });
  }

  const quiz = {
    id: randomUUID(),
    title: title.trim(),
    subject: subject.trim(),
    questions: questions.map(q => ({
      question: q.question.trim(),
      options: q.options.map(option => option.trim()),
      correct: q.correct
    }))
  };

  quizzes.push(quiz);

  res.status(201).json({
    message: "¡Reto creado correctamente!",
    quiz: {
      id: quiz.id,
      title: quiz.title,
      subject: quiz.subject,
      questionCount: quiz.questions.length
    }
  });
});

// Enviar respuestas y calcular puntos en el servidor
app.post("/api/quizzes/:id/attempts", (req, res) => {
  const quiz = quizzes.find(q => q.id === req.params.id);

  if (!quiz) {
    return res.status(404).json({
      error: "No se encontró el reto."
    });
  }

  const { studentId, answers } = req.body;

  if (
    typeof studentId !== "string" ||
    !/^[a-zA-Z0-9_-]{1,50}$/.test(studentId) ||
    !Array.isArray(answers) ||
    answers.length !== quiz.questions.length ||
    !answers.every(Number.isInteger)
  ) {
    return res.status(400).json({
      error: "El estudiante o las respuestas no son válidos."
    });
  }

  // Cada respuesta debe corresponder a una opción existente
  const validAnswers = answers.every(
    (answer, index) =>
      answer >= 0 &&
      answer < quiz.questions[index].options.length
  );

  if (!validAnswers) {
    return res.status(400).json({
      error: "Una respuesta está fuera de las opciones disponibles."
    });
  }

  let correctCount = 0;

  quiz.questions.forEach((question, index) => {
    if (answers[index] === question.correct) {
      correctCount++;
    }
  });

  const earned = correctCount * 10;
  const profile = getProfile(studentId);

  profile.points += earned;
  profile.completed++;

  const attempt = {
    quizId: quiz.id,
    title: quiz.title,
    correct: correctCount,
    total: quiz.questions.length,
    earned,
    date: new Date().toISOString()
  };

  profile.attempts.push(attempt);

  res.json({
    message: "Intento registrado.",
    result: attempt,
    profile: {
      points: profile.points,
      level: getLevel(profile.points),
      completed: profile.completed,
      pointsToNextLevel: 100 - (profile.points % 100)
    }
  });
});

// Consultar estadísticas de un estudiante
app.get("/api/students/:studentId/stats", (req, res) => {
  const { studentId } = req.params;

  if (!/^[a-zA-Z0-9_-]{1,50}$/.test(studentId)) {
    return res.status(400).json({
      error: "ID de estudiante inválido."
    });
  }

  const profile = getProfile(studentId);

  res.json({
    studentId,
    points: profile.points,
    level: getLevel(profile.points),
    completed: profile.completed,
    attempts: profile.attempts
  });
});

// Manejo básico de rutas inexistentes
app.use((req, res) => {
  res.status(404).json({
    error: "Ruta no encontrada."
  });
});

app.listen(PORT, "0.0.0.0", () => {
  console.log(`Reto Escolar activo en http://localhost:${PORT}`);
});
