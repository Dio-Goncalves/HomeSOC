# HomeSOC
This project consists on building your own SOC at home. This is a pilot project that will serve as foundation for the next version I have already planned. I've built this in order to learn more about each of the utilized and how they can work together and behave as an infrastructure.
I'll not only focus on the SIEM but also on setting up the machines and network. Using this repository as reference, and maybe some more investigation on the reader's end for solving specific issues that may appear, it should be possible to replicate this lab.
This is my first ever project of this kind and also first ever GitHub repository, so any feedback is welcome!

## Repository Structure
You can find below the structure of this repository with direct links for each part

## Lessons Learned
**Documentation is key**. It was something I had very clear and that paid a lot of attention to. But on a project this big, there were times that due to time constraints from the trial software (mostly Splunk with its 2 month free trial) and also due to the fact that I have a job and have a life outside of this, I didn't document as well as I should. There were times that, while I was assembling the lab and running trials, I didn't document every step in order to avoid repeating myself or documenting unnecessary stuff in case what I was doing didn't work. Later, this proved to be a big mistake and made me lose even more time when building this repository and referencing to some of the things I did previously. This is also the main reason why I want to migrate the project to a physical infrastructure and to only use fully open-source software, I want a consistent structure without the volatility of a Virtual Machine that is also stealing resources from the host PC or the limited free timed trials of proprietary software. Having a physical structure and open-source software will allow me to work in a timely and modular way, having time to precisely document each step and also building the repository as I go.
This was a big learning process, not only in technical terms and learning the tools but also on the time management side of things and knowing what to expect.

## Next Steps
 - Migrate the lab to dedicated physical servers;
 - Use open-source software only;
 - Dive deeper into network segmentation;
 - Run packet captures and analyze them with Wireshark;
 - Containerization with docker and kubernetes for container management;
 - Implement Sigma rules;
 - Implement Honeypots;
 - Implement MISP for threat intelligence;
 - Implement an EDR;
 - Implement the usage of a SOAR;
 - Automation of Threat Intelligence through STIX/TAXII with a local LLM for automatic report generation;
 - Implement some sort of ticketing platform to simulate a more realistic workflow (such as JIRA).

## ⚠️⚠️DISCLAIMER⚠️⚠️  
This is a live repository, even though the lab is fully built, the repository is still far from being finished.
