<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Cybersecurity Portfolio</title>
  <link href="https://fonts.googleapis.com/css2?family=Poppins:wght@300;400;600&display=swap" rel="stylesheet">
  <style>
    * {
      margin: 0;
      padding: 0;
      box-sizing: border-box;
      font-family: 'Poppins', sans-serif;
    }
    body {
      background: #1a1a1a;
      color: #e0e0e0;
      line-height: 1.6;
    }
    header {
      background: #0d0d0d;
      color: #e0e0e0;
      padding: 2rem 0;
      text-align: center;
    }
    header h1 {
      font-size: 2.5rem;
      margin-bottom: 0.5rem;
    }
    header p {
      font-size: 1.2rem;
      color: #aaa;
    }
    nav {
      background: #141414;
      display: flex;
      justify-content: center;
      gap: 2rem;
      padding: 1rem 0;
    }
    nav a {
      color: #e0e0e0;
      text-decoration: none;
      font-weight: 500;
      transition: color 0.3s;
    }
    nav a:hover {
      color: #00adb5;
    }
    section {
      padding: 4rem 10%;
    }
    section h2 {
      font-size: 2rem;
      margin-bottom: 1.5rem;
      text-align: center;
      color: #fff;
    }
    .about p {
      max-width: 700px;
      margin: auto;
      text-align: center;
      font-size: 1.1rem;
      color: #ccc;
    }
    .projects {
      display: grid;
      grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
      gap: 2rem;
      margin-top: 2rem;
    }
    .project-card {
      background: #262626;
      border-radius: 12px;
      box-shadow: 0 4px 10px rgba(0,0,0,0.4);
      padding: 1.5rem;
      transition: transform 0.3s;
    }
    .project-card:hover {
      transform: translateY(-5px);
    }
    .project-card h3 {
      margin-bottom: 0.5rem;
      color: #fff;
    }
    .skills ul, .tools ul, .certs ul {
      list-style: none;
      display: flex;
      flex-wrap: wrap;
      justify-content: center;
      gap: 1rem;
    }
    .skills li, .tools li, .certs li {
      background: #333;
      color: #e0e0e0;
      padding: 0.5rem 1rem;
      border-radius: 20px;
      font-size: 0.95rem;
      border: 1px solid #444;
    }
    .contact {
      text-align: center;
    }
    .contact a {
      display: inline-block;
      margin: 0.5rem;
      text-decoration: none;
      color: #00adb5;
      font-weight: 600;
      transition: color 0.3s;
    }
    .contact a:hover {
      color: #007780;
    }
    footer {
      background: #0d0d0d;
      color: #777;
      text-align: center;
      padding: 1.5rem;
      font-size: 0.9rem;
    }
  </style>
</head>
<body>
  <header>
    <h1>Abiodun Victor Taiwo</h1>
    <p>Pentester | Security Researcher | Cybersecurity Enthusiast</p>
  </header>

  <nav>
    <a href="#about">About</a>
    <a href="#labs">Labs & Writeups</a>
    <a href="#tools">Tools</a>
    <a href="#certs">Certifications</a>
    <a href="#contact">Contact</a>
  </nav>

  <section id="about" class="about">
    <h2>About Me</h2>
    <p>Hello! I'm a cybersecurity professional specializing in penetration testing and ethical hacking. I enjoy exploring attack surfaces, exploiting vulnerabilities, and sharing my knowledge.</p>
  </section>

  <section id="labs">
    <h2>Labs & Writeups</h2>
    <div class="projects">
      <div class="project-card">
        <h3>HackTheBox</h3>
        <p>My writeups and solved boxes are documented here. 
          <a href="/htb/">View Writeups</a>
        </p>
      </div>
      <div class="project-card">
        <h3>TryHackMe</h3>
        <p>Learning-based labs and walkthroughs are available here. 
          <a href="/thm/">View Writeups</a>
        </p>
      </div>
      <div class="project-card">
        <h3>Blog Writeups</h3>
        <p>Detailed walkthroughs of labs, exploits, and real-world security concepts. 
          <a href="https://medium.com/@3xploit">Visit Blog</a>
        </p>
      </div>
    </div>
  </section>

  <!-- Skills Section -->
<section id="skills" class="p-8 max-w-5xl mx-auto">
  <h2 class="text-2xl font-bold text-green-400 mb-8 text-center">Skills</h2>
  <div class="flex flex-wrap gap-4 justify-center">
    <span class="px-5 py-2 rounded-full bg-black text-gray-300 border border-gray-700 shadow-md hover:shadow-green-500/30 hover:text-green-400 transition">Web Application Testing</span>
    <span class="px-5 py-2 rounded-full bg-black text-gray-300 border border-gray-700 shadow-md hover:shadow-green-500/30 hover:text-green-400 transition">Active Directory Exploitation</span>
    <span class="px-5 py-2 rounded-full bg-black text-gray-300 border border-gray-700 shadow-md hover:shadow-green-500/30 hover:text-green-400 transition">Linux Privilege Escalation</span>
    <span class="px-5 py-2 rounded-full bg-black text-gray-300 border border-gray-700 shadow-md hover:shadow-green-500/30 hover:text-green-400 transition">Windows Privilege Escalation</span>
    <span class="px-5 py-2 rounded-full bg-black text-gray-300 border border-gray-700 shadow-md hover:shadow-green-500/30 hover:text-green-400 transition">Networking & Protocol Analysis</span>
    <span class="px-5 py-2 rounded-full bg-black text-gray-300 border border-gray-700 shadow-md hover:shadow-green-500/30 hover:text-green-400 transition">Exploit Development Basics</span>
  </div>
</section>

<!-- Tools Section -->
<section id="tools" class="p-8 max-w-6xl mx-auto">
  <h2 class="text-2xl font-bold text-green-400 mb-8 text-center">Tools</h2>
  <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-8">
    <div class="bg-gray-900 p-6 rounded-xl border border-gray-700 shadow-md hover:shadow-green-500/20 hover:-translate-y-2 transition transform">
      <h3 class="text-lg font-semibold text-green-400 mb-2">Tool 1</h3>
      <p class="text-gray-400 mb-4">A custom scanner for XYZ.</p>
      <a href="https://github.com/yourusername/tool1" class="text-green-400 hover:underline">View on GitHub</a>
    </div>
    <div class="bg-gray-900 p-6 rounded-xl border border-gray-700 shadow-md hover:shadow-green-500/20 hover:-translate-y-2 transition transform">
      <h3 class="text-lg font-semibold text-green-400 mb-2">Tool 2</h3>
      <p class="text-gray-400 mb-4">Automates recon and enumeration.</p>
      <a href="https://github.com/yourusername/tool2" class="text-green-400 hover:underline">View on GitHub</a>
    </div>
    <!-- Add more tool cards as needed -->
  </div>
  <p class="text-center mt-6">
    <a href="https://github.com/yourusername" class="text-green-400 hover:underline">See more on GitHub →</a>
  </p>
</section>


  <section id="certs" class="certs">
    <h2>Certifications</h2>
    <ul>
      <li>CRTA (Certified Red Team Analyst)</li>
      <li>CAP</li>
      <li>CNSP</li>
    </ul>
  </section>

  <section id="contact" class="contact">
    <h2>Contact Me</h2>
    <p>You can reach me through the following platforms:</p>
    <a href="mailto:victorolatunde656@gmail.com">Email</a>
    <a href="https://github.com/3XP-LOIT">GitHub</a>
    <a href="https://www.linkedin.com/in/victor-abiodun-a970712a5?">LinkedIn</a>
  </section>

  <footer>
    <p>&copy; 2025 Abiodun Victor Taiwo. All rights reserved.</p>
  </footer>
</body>
</html>
