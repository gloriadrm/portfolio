# 🌐 Portfolio Personal

El objetivo de este sitio web es centralizar mi experiencia, competencias técnicas y proyectos, mostrando no solo el resultado final, sino también las decisiones de diseño y el aprendizaje adquirido durante cada desarrollo.

## 🚀 Tecnologías utilizadas

- Astro
- Tailwind CSS
- daisyUI

## 📂 Estructura del proyecto

```
gloria-portfolio/
│
├── src/
│   ├── pages/              → index, sobre-mi, proyectos/ (index + [...slug])
│   ├── layouts/             → layout compartido (nav, footer, Google Analytics)
│   ├── components/          → tarjetas de proyecto, tags, badges, GoogleAnalytics...
│   ├── content/               → proyectos/ y sobre-mi/ (Markdown)
│   ├── assets/proyectos/       → imágenes estáticas del cuerpo de cada caso de estudio
│   ├── data/                     → site.json
│   ├── lib/                       → analytics.ts (helpers de tracking)
│   └── styles/                     → tema daisyUI y estilos globales
├── public/
│   ├── assets/                → CV descargable
│   ├── favicon.svg
│   └── proyectos/<slug>/        → memoria.pdf y animaciones grandes de cada proyecto
└── .github/workflows/            → despliegue automático a GitHub Pages
```

## ✨ Funcionalidades

- Diseño responsive.
- Navegación mediante menú superior con desplegable de proyectos.
- Animaciones al hacer scroll.
- Timeline de próximos proyectos y acordeones en los casos de estudio.
- Comparador interactivo de decisiones técnicas (elegido vs. descartado).
- Toggle de tema claro/oscuro.
- Contenido separado del diseño (Markdown y JSON), editable sin tocar el código.
- Enlaces a GitHub, LinkedIn y contacto.

## 📌 Secciones

- Sobre mí
- Experiencia profesional
- Competencias técnicas
- Portfolio de proyectos
- Contacto

## 🎯 Objetivo

Este portfolio continuará evolucionando. Cada proyecto nace para profundizar en una tecnología o resolver un problema concreto. El objetivo es comprender cómo y por qué utilizar cada herramienta, documentando el proceso de aprendizaje y las decisiones técnicas adoptadas durante el desarrollo.

## ⚙️ Ingeniería del portfolio

El portfolio no solo documenta proyectos; también documenta cómo se documentan.

Para evitar inconsistencias entre proyectos se ha diseñado un flujo reproducible que separa:

- generación de la memoria técnica;
- generación de la página web;
- revisión editorial;
- publicación.

Este enfoque permite mantener una única fuente de verdad para el contenido y reutilizar el mismo proceso en cualquier proyecto futuro.

## 📬 Contacto

📧 **gloriadrm@hotmail.com**

🔗 LinkedIn
https://www.linkedin.com/in/gloria-del-rio-marquez

💻 GitHub
https://github.com/gloriadrm

---

Instrucciones de instalación, desarrollo local y despliegue: ver [DESARROLLO.md](./DESARROLLO.md)
