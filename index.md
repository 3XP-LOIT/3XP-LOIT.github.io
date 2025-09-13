<!DOCTYPE html>
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
    .skills ul, .tools ul, .certs ul, .downloads ul {
      list-style: none;
      display: flex;
      flex-wrap: wrap;
      justify-content: center;
      gap: 1rem;
    }
    .skills li, .tools li, .certs li, .downloads li {
      background: #333;
      color: #e0e0e0;
      padding: 0.5rem 1rem;
      border-radius: 20px;
      font-size: 0.95rem;
      border: 1px solid #444;
    }
    .downloads a {
      color: #00adb5;
      text-decoration: none;
      font-weight: 600;
    }
    .downloads a:hover {
      color: #007780;
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
    <h1>My Cybersecurity Portfolio</h1>
    <p>Pentester | Security Researcher | Cybersecurity Enthusiast</p>
  </header>

  <nav>
    <a href="#about">About</a>
    <a href="#labs">Labs & Writeups</a>
    <a href="#tools">Tools</a>
    <a href="#certs">Certifications</a>
    <a href="#downloads">Downloads</a>
    <a href="#contact">Contact</a>
  </nav>

  <section id="about" class="about">
    <h2>About Me</h2>
    <p>Hello! I'm a cybersecurity professional specializing in penetration testing and ethical hacking. I enjoy exploring attack surfaces, exploiting vulnerabilities, and sharing my knowledge through labs and writeups.</p>
  </section>

  <section id="labs">
    <h2>Labs & Writeups</h2>
    <div class="projects">
      <div class="project-card">
        <h3>HackTheBox</h3>
        <p>My profile showcasing solved boxes and challenges. <a href="#">View Profile</a></p>
      </div>
      <div class="project-card">
        <h3>TryHackMe</h3>
        <p>Learning-based labs and certifications. <a href="#">View Profile</a></p>
      </div>
      <div class="project-card">
        <h3>Blog Writeups</h3>
        <p>Detailed walkthroughs of labs, exploits, and real-world security concepts. <a href="#">Visit Blog</a></p>
      </div>
    </div>
  </section>

  <section id="tools" class="tools">
    <h2>Tools & Skills</h2>
    <ul>
      <li>Burp Suite</li>
      <li>Nmap</li>
      <li>Metasploit</li>
      <li>Wireshark</li>
      <li>BloodHound</li>
      <li>Active Directory Attacks</li>
      <li>Web App Pentesting</li>
      <li>Exploit Development</li>
    </ul>
  </section>

  <section id="certs" class="certs">
    <h2>Certifications</h2>
    <ul>
      <li>OSCP (Planned)</li>
      <li>eJPT</li>
      <li>CEH</li>
    </ul>
  </section>

  <section id="downloads" class="downloads">
    <h2>Downloads</h2>
    <ul>
      <li><a href="cv.pdf" download>Download My CV</a></li>
      <li><a href="sample_pentest_report.pdf" download>Sample Pentest Report</a></li>
    </ul>
  </section>

  <section id="contact" class="contact">
    <h2>Contact Me</h2>
    <p>You can reach me through the following platforms:</p>
    <a href="mailto:youremail@example.com">Email</a>
    <a href="https://github.com/yourusername">GitHub</a>
    <a href="https://linkedin.com/in/yourprofile">LinkedIn</a>
  </section>

  <footer>
    <p>&copy; 2025 Your Name. All rights reserved.</p>
  </footer>
</body>
</html>
