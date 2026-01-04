# README Restructuring Instructions for Project Suite

## Objective

Restructure project README files for improved consistency, human scannability, and documentation organization across the project suite using action-assertive headings and bullet-point content structure.

## README Structure Requirements

### Table of Contents Structure

1. **Main heading** - Project name as H1 heading
2. **Concise secondary heading** - H2 heading with brief, scannable project description (avoid verbose technical details)
3. **Project detail paragraph** - Comprehensive paragraph including technical details moved from verbose heading, plus repository link
4. **What Problem This Project Solves** - Problem statement with introductory sentence and bullet points
5. **What This Project Does** - Solution approach with introductory sentence and bullet points  
6. **What This Project Changes** - Resources created/managed and functional changes with introductory sentence
7. **Quick Start** - Basic usage with references to docs subdirectory
8. **AWS Well-Architected Framework** - Assessment against 6 pillars
9. **Technologies Used** - Comprehensive list in three-column table format (Technology | Purpose | Implementation)
10. **Copyright** - Single line matching LICENSE file

### Content Standards

- **Headings**: Concise and scannable - move verbose technical details to paragraph content
- **Action-assertive format**: Use "What Problem This Project Solves", "What This Project Does", "What This Project Changes"
- **Bullet points**: Convert paragraph content to scannable bullet points for improved readability
- **Introductory sentences**: Each main section starts with brief introductory sentence followed by detailed bullets
- **Complete information**: Maintain all technical details while improving scannability
- **Consistent structure**: All projects follow identical heading and content organization patterns

## Main Section Content Structure

### What Problem This Project Solves
- Brief introductory sentence summarizing the core problem
- Bullet points detailing specific pain points and challenges
- Focus on problems that exist without this solution

### What This Project Does  
- Brief introductory sentence summarizing the solution approach
- Bullet points detailing specific capabilities and features
- Focus on how the project addresses the identified problems

### What This Project Changes
- Brief introductory sentence summarizing overall impact
- **Resources Created/Managed** subsection listing AWS resources
- **Functional Changes** subsection describing operational capabilities enabled

## Quick Start Requirements

- Move prerequisites to `/docs/prerequisites.md`
- Move troubleshooting to `/docs/troubleshooting.md`
- Reference scripts directory
- Keep basic usage flow only
- Include contextual references to all documents in `/docs` folder
- Each docs reference must provide context explaining what the document contains

## AWS Well-Architected Framework Assessment

### Assessment Process

1. **Individual pillar assessment** - Consider each pillar against project characteristics
2. **Overall project review** - Cross-check with holistic project view
3. **Reconcile perspectives** - Combine insights from both approaches
4. **Include only aligned pillars** - Omit pillars without demonstrable alignment

### Six Pillars to Evaluate

- **Operational Excellence** - Automation, monitoring, operational procedures
- **Security** - Encryption, access controls, authentication, governance
- **Reliability** - Backup strategies, error recovery, fault tolerance
- **Performance Efficiency** - Auto-scaling, optimization, efficient resource usage
- **Cost Optimization** - Intelligent storage, pay-per-use, cost visibility
- **Sustainability** - Efficient resource utilization, reduced operational overhead

### Assessment Criteria

- Focus on demonstrable characteristics, not aspirational claims
- Consider serverless/managed service usage
- Evaluate automation and operational tooling
- Assess security practices and access controls
- Review cost optimization strategies
- Consider environmental impact reduction

## Technologies Used Section

### Requirements

- Comprehensive list including all technologies
- **Kiro CLI with Claude must be listed first** as the primary development tool
- Assume nothing - include Bash, IaC, Parameter Store, all AWS resource types, Git, etc.
- Prioritize complex/significant technologies over common ones
- Target search optimization for accurate key terms
- **Use three-column table format**: Technology | Purpose | Implementation
- Implementation column should provide specific technical details about how each technology is used

### Technology Categories to Include

- AWS services (all used services)
- Infrastructure as Code tools
- Scripting languages
- Development tools
- Security technologies
- Automation tools
- Data processing tools

## Documentation Structure

### Create /docs Directory

Required subdocuments:

- `prerequisites.md` - Detailed requirements moved from README
- `troubleshooting.md` - Comprehensive troubleshooting guide
- `tags.md` - Resource tagging documentation (if applicable)
- Additional subdocs as needed for project-specific content

### Content Migration

- Move verbose content from README to appropriate subdocs
- Maintain references in README to subdocs
- Optimize README for scanning while preserving completeness in subdocs

## Implementation Process

1. **Create concise secondary heading**
   - Remove verbose technical details from H2 heading
   - Keep heading brief and scannable
   - Move detailed technical information to paragraph content

2. **Structure main content sections**
   - Use action-assertive headings: "What Problem This Project Solves", "What This Project Does", "What This Project Changes"
   - Add brief introductory sentence to each section
   - Convert paragraph content to bullet points for scannability
   - Maintain all technical details while improving readability

3. **Assess project against AWS Well-Architected Framework**
   - Individual pillar evaluation
   - Overall project cross-check
   - Reconcile findings

4. **Create /docs directory structure**
   - Prerequisites documentation
   - Troubleshooting guide
   - Additional project-specific docs

5. **Verify consistency across projects**
   - Check all projects follow identical structure
   - Ensure consistent heading format and bullet-point organization
   - Validate improved scannability while preserving completeness

## Quality Checklist

- [ ] Main heading followed by concise secondary heading
- [ ] Secondary heading is brief and scannable (technical details moved to paragraph)
- [ ] Project detail paragraph includes comprehensive technical information and repository link
- [ ] "What Problem This Project Solves" has introductory sentence and bullet points
- [ ] "What This Project Does" has introductory sentence and bullet points
- [ ] "What This Project Changes" has introductory sentence and subsections
- [ ] All paragraph content converted to scannable bullet points
- [ ] Quick Start references docs subdirectory
- [ ] AWS Well-Architected includes only aligned pillars
- [ ] Technologies list is comprehensive and uses three-column table format (Technology | Purpose | Implementation)
- [ ] Kiro CLI with Claude listed as first technology
- [ ] Implementation column provides specific technical details for each technology
- [ ] Copyright matches LICENSE file
- [ ] /docs directory contains moved content
- [ ] All references and links work correctly
- [ ] Content optimized for human scannability with consistent structure across projects
