<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Cybersecurity Portfolio</title>
  <link href="https://fonts.googleapis.com/css2?family=Poppins:wght@300;400;600&display=swap" rel="stylesheet">
  <link href="https://cdn.jsdelivr.net/npm/tailwindcss@2.2.19/dist/tailwind.min.css" rel="stylesheet">
  <style>
  /* Skills */
  .skills {
    display: flex;
    flex-wrap: wrap;
    gap: 12px;
    justify-content: center;
  }
  .skill-badge {
    background: #111;            /* deep black */
    color: #9ca3af;             /* Tailwind gray-400 */
    padding: 8px 14px;
    border-radius: 20px;
    border: 1px solid #1f2937;  /* Tailwind gray-800 border */
    font-size: 14px;
    transition: all 0.2s ease;
  }
  .skill-badge:hover {
    background: #1f2937;        /* Tailwind gray-800 */
    color: #10b981;             /* Tailwind green-500 */
  }

  /* Tools */
  .tools {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
    gap: 20px;
    margin-top: 20px;
  }
  .tool-card {
    background: #111;            /* dark background */
    padding: 20px;
    border-radius: 10px;
    border: 1px solid #1f2937;  /* subtle dark border */
    box-shadow: 0 2px 6px rgba(0,0,0,0.6);
    transition: all 0.2s ease;
  }
  .tool-card:hover {
    border-color: #10b981;      /* green glow on hover */
    transform: translateY(-4px);
  }
  .tool-card h3 {
    margin: 0 0 10px;
    color: #10b981;             /* green title */
    font-size: 18px;
  }
  .tool-card p {
    color: #d1d5db;             /* Tailwind gray-300 */
    margin-bottom: 12px;
    font-size: 14px;
  }
  .tool-card a {
    color: #10b981;             /* green links */
    font-weight: 500;
    text-decoration: none;
  }
  .tool-card a:hover {
    text-decoration: underline;
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
 <section id="skills" class="p-8 max-w-4xl mx-auto">
   <h2 class="text-2xl font-bold text-green-400 mb-6 text-center">Skills</h2>
   <div class="flex flex-wrap gap-3 justify-center">
     <span class="bg-gray-800 text-gray-200 px-4 py-2 rounded-full border border-gray-700">Web Application Testing</span>
     <span class="bg-gray-800 text-gray-200 px-4 py-2 rounded-full border border-gray-700">Active Directory Exploitation</span>
     <span class="bg-gray-800 text-gray-200 px-4 py-2 rounded-full border border-gray-700">Linux Privilege Escalation</span>
     <span class="bg-gray-800 text-gray-200 px-4 py-2 rounded-full border border-gray-700">Windows Privilege Escalation</span>
     <span class="bg-gray-800 text-gray-200 px-4 py-2 rounded-full border border-gray-700">Networking & Protocol Analysis</span>
     <span class="bg-gray-800 text-gray-200 px-4 py-2 rounded-full border border-gray-700">Exploit Development Basics</span>
   </div>
 </section>


  <!-- Tools Section -->
  <section id="tools" class="p-8 max-w-5xl mx-auto">
  <h2 class="text-2xl font-bold text-green-400 mb-6 text-center">Tools</h2>
  <div class="grid grid-cols-1 md:grid-cols-2 gap-6">
    <div class="bg-gray-800 p-5 rounded-lg shadow hover:shadow-lg transition">
      <h3 class="text-xl font-semibold text-gray-100">Tool 1</h3>
      <p class="text-gray-400 mb-2">A custom scanner for XYZ.</p>
      <a href="https://github.com/yourusername/tool1" class="text-green-400 hover:underline">View on GitHub</a>
    </div>
    <div class="bg-gray-800 p-5 rounded-lg shadow hover:shadow-lg transition">
      <h3 class="text-xl font-semibold text-gray-100">Tool 2</h3>
      <p class="text-gray-400 mb-2">Automates recon and enumeration.</p>
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
