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
      color: #ccc;
    }
    section {
      padding: 4rem 10%;
    }
    section h2 {
      font-size: 2rem;
      margin-bottom: 1.5rem;
      text-align: center;
      color: #f1f1f1;
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
      background: #2b2b2b;
      color: #e0e0e0;
      padding: 0.6rem 1.2rem;
      border-radius: 20px;
      font-size: 0.95rem;
      border: 1px solid #444;
      transition: background 0.3s, transform 0.3s;
    }
    .skills li:hover, .tools li:hover, .certs li:hover {
      background: #3a3a3a;
      transform: translateY(-3px);
    }
    .contact {
      text-align: center;
    }
    .contact a {
      display: inline-block;
      margin: 0.5rem;
      text-decoration: none;
      color: #bbb;
      font-weight: 600;
      transition: color 0.3s;
    }
    .contact a:hover {
      color: #fff;
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
    <a href="#skills">Skills</a>
    <a href="#tools">Tools</a>
    <a href="#certs">Certifications</a>
    <a href="#contact">Contact</a>
  </nav>

  <section id="about" class="about">
    <h2>About Me</h2>
    <p>Hello! I'm a cybersecurity professional specializing in penetration testing and ethical hacking. I enjoy exploring attack surfaces, exploiting vulnerabilities, and giving back to the community.</p>
  </section>

  <section id="labs">
    <h2>Labs & Writeups</h2>
    <div class="projects">
      <div class="project-card">
        <h3>HackTheBox</h3>
        <p>HackTheBox walkthroughs.<a href="/htb">Access Here</a></p>
      </div>
      <div class="project-card">
        <h3>TryHackMe</h3>
        <p>TryHackMe Writeups<a href="/thm">Access Here</a></p>
      </div>
      <div class="project-card">
        <h3>Blog Writeups</h3>
        <p>Detailed walkthroughs of labs, exploits, and security concepts. <a href="https://medium.com/@3xploit">Blog</a></p>
      </div>
    </div>
  </section>

  <section id="skills" class="skills">
    <h2>Skills</h2>
    <ul>
      <li>Web App Pentesting</li>
      <li>Active Directory Exploitation</li>
      <li>Cloud Pentesting</li>
      <li>Python and Bash Scripting</li>
      <li>Burpsuite</li>
      <li>Networking</li>
    </ul>
  </section>

  <section id="tools" class="tools">
    <h2>Tools</h2>
    <ul>
      <li><a href="https://github.com/3XP-LOIT/Password_Checker" style="color:#bbb;text-decoration:none;">Password Checker</a></li>
      <li><a href="https://github.com/3XP-LOIT/Network_scanner" style="color:#bbb;text-decoration:none;">Network Scanner</a></li>
      <li><a href="https://github.com/3XP-LOIT" style="color:#bbb;text-decoration:none;">More on GitHub</a></li>
    </ul>
  </section>

  <section id="certs" class="certs">
    <h2>Certifications</h2>
    <ul>
      <li>CRTA</li>
      <li>CAP</li>
      <li>CNSP</li>
    </ul>
  </section>

  <section id="contact" class="contact">
    <h2>Contact Me</h2>
    <p>You can reach me through the following platforms:</p>
    <a href="mailto:victorolatunde656@gmail.com">Email</a>
    <a href="https://github.com/3XP-LOIT">GitHub</a>
    <a href="https://www.linkedin.com/in/victor-abiodun-a970712a5/?">LinkedIn</a>
  </section>

  <footer>
    <p>&copy; 2025 Abiodun Victor Taiwo. All rights reserved.</p>
  </footer>
</body>
</html>
