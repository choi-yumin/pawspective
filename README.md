# 🐾 Pawspective

An interactive, responsive pseudo-3D web experience built with **Svelte** and unique Javascript library called **Zdog**. Explore unique environments, switch depths, and dive into perspective shifts to animals' internal worlds that are often not what as insignificant as one might assume.

**Students:** Amanbay Makhabbat (ID: 20240935) and Yu Min Choi (ID: 20220899)
**Department:** Industrial Design, KAIST  
**Contact:** [mako1004@kaist.ac.kr](mailto:mako1004@kaist.ac.kr)  
**Links:** [Website](https://pawspective-1.netlify.app/) | [Git Repository](https://git.prototyping.id/20220899/pawspective.git) | [Video Demo](https://www.youtube.com/watch?v=Sbo288wAwSE)

## Core Concept

The idea originated from the idea that humans either idealize the freedom of animals or have incorrect and stereotypical assumptions about that are not necessarily true. We decided to make this playful and explorative website where one can chat and explore the interactives these animals can provide.

**Some of the base ideas include:**
Birds - idealized for freedom - actually constantly stressed
Bees - seems annoying and scary - are actually diligent and cute
Chicken - widely considered stupid - but are surprisingly smart
Blobfish - widely accepted as ugliest fish - but is actually normal looking in its own environment
Sloth - it looks lazy - but it actually just biologically lacks stamiina, they would also like to play all they which they can't do

## Key Features

- **Pseudo-3D Environments:** Rendered entirely using vector-based geometric structures powered by `zdog`.
- **Dual-Depth Ecosystems:** Toggle fluidly between a vibrant surface land environment filled with trees and foliage, and a stylized deep-sea underwater simulation.
- **Dynamic Radial Interaction Menu:** Contextual action cards pop up on pointer interactions to let users interact with the environment models.
- **Interactive Chat Session History:** View and manage localized contextual session histories through an overlay drawer panel.
- **Smooth Gesture Controls:** Fully immersive pan, drag, and click actions engineered to operate uniformly across desktop browsers and mobile touchpoints.

---

## Tech Stack

- **Frontend Framework:** Svelte (Svelte 5 Runes architecture)
- **3D Vector Engine:** [Zdog](https://zzz.dog/) (Round, flat, designer-friendly pseudo-3D engine)
- **Animation Engine:** GSAP for manipulating and animating Zdog components
- **Hosting & Backend:** Netlify
- **AI Integration:** OPENAI API

# Resources Used

Inspiration and reference design from: https://codepen.io/dabalog/pen/xxqjNyX

## AI Usage

Claude and Gemini were mainly used for the creation of the website.

Example prompts:

- `Modify the Sloth.svelte file to match the Bee.svelte layout: move the description panel to the bottom, place the chat interface in the bottom-right corner, and remove the radial interaction menu.`
- `Refine the sloth's body proportions and anatomy to appear more natural. Also, disable user-controlled rotation. Use the provided reference image and code-output screenshot as guidance.`
