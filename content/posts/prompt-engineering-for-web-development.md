---
title: "Prompt Engineering for Web Development"
date: 2024-03-21
draft: false
---

Effective prompt engineering can dramatically improve the quality of AI-generated code for web development projects. This article explores key strategies to maximize your results when working with AI coding assistants.

## Defining Clear Objectives and Requirements

When crafting prompts for web development, it is crucial to define clear objectives and requirements. Begin by specifying the desired outcome, such as generating a specific code snippet, explaining a concept, or providing guidance on a particular development task. Be precise in your expectations and include any constraints or limitations that the AI model should adhere to.

<table>
<thead>
<tr>
<th>Do</th>
<th>Don't</th>
</tr>
</thead>
<tbody>
<tr>
<td>✓ "Generate an HTML and CSS code snippet for a responsive navigation menu that collapses into a hamburger icon on mobile devices."</td>
<td>✗ "How do I create a responsive navigation menu?"</td>
</tr>
</tbody>
</table>

> 💡 **Tip:**  
> A solid knowledge of web technologies will help tremendously when prompting. Take some time to learn the fundamentals!
>
> The idea is to split the project into small, achievable tasks, just like a developer would if they were doing it themselves. The resulting code will be better, and this approach has several positive side-effects: the result is easier to test and adjust (the mistakes are also less expensive). Tasks can be sent in parallel if they are not directly correlated, meaning you can spend less time waiting for a response.

## Using Domain-Specific Language and Terminology

Employ technical terms, programming languages, frameworks, and libraries that are relevant to your task. This helps the model comprehend the context and generate more precise and relevant responses.

<table>
<thead>
<tr>
<th>Do</th>
<th>Don't</th>
</tr>
</thead>
<tbody>
<tr>
<td>✓ "Using Font Awesome and SweetAlert 2, build a 'Share' button that displays a popup with the link of the page, with clickable icons for Instagram, Facebook, etc."</td>
<td>✗ "Write a way to display a popup when the share button is clicked with the link and choice between Facebook, Instagram, etc."</td>
</tr>
</tbody>
</table>

> 💡 **Tip:**  
> Don't hesitate to prompt a free/cheap model like GPT-3.5 to get the frameworks and libraries: "Give me five relevant frameworks and libraries to achieve XYZ"

## The Single HTML File Trick

### Or How to Quickly Build a Basic UI and Connect it to Your Backend

When prototyping in the initial phase of a task, simply requesting a single HTML file will give you an output that is self-contained, easily testable UI-only mockup.

You can then modify what's necessary, explain your backend API, and prompt to "Replace the static data with data from the backend API."

## API First, Frontend Later

Sometimes, it's easier to build the backend side first, give a sample of the JSON response to the model, and prompt to build a beautiful dashboard using the appropriate libraries.

<table style="table-layout: fixed; width: 100%;">
<thead>
<tr>
<th width="50%">Do</th>
<th width="50%">Don't</th>
</tr>
</thead>
<tbody>
<tr>
<td>✓ "I have a JSON API at endpoint mywebsite.xyz/api that returns:<br>[{<span style="font-style: italic;">"id": "abc", "content": "sample content"</span>},...]<br>Display this into a beautiful list of cards."</td>
<td>✗ "Given a list of elements, write the HTML code to display a list of elements."</td>
</tr>
</tbody>
</table> 