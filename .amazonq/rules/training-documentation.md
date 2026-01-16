# Training Documentation Rules for Senior Java Developer

## Documentation Standards

### Technical Depth Requirements
- **Internal Implementation**: Always explain underlying Spring Security classes, interfaces, and mechanisms
- **Architecture Context**: Show how components fit into the overall Spring Security architecture
- **Alternative Approaches**: Present multiple implementation options with trade-offs
- **Real-world Scenarios**: Include common use cases beyond basic examples

### Content Structure
- **Concise but Complete**: No redundant explanations, focus on essential technical details
- **Code-to-Concept Mapping**: Connect configuration code to actual Spring classes and processes
- **Class Type Clarity**: Always specify class types for fluent API parameters (e.g., `HttpSecurity http`, `AuthorizeHttpRequestsConfigurer requests`)
- **Extensibility Focus**: Show how to extend or customize default behavior
- **Performance Implications**: Mention when choices impact performance or scalability

### Coverage Requirements
When documenting Spring Security features, include:

1. **Core Classes**: Primary interfaces, implementations, and their relationships
2. **Configuration Options**: Available alternatives to demonstrated approach
3. **Extension Points**: How to customize or replace default components
4. **Integration Patterns**: How feature works with other Spring Security components
5. **Common Pitfalls**: Typical mistakes and how to avoid them

### Interaction Protocol
- **Ask for Clarification**: If requirements, scope, or technical level is unclear, ask specific questions
- **Validate Understanding**: Confirm interpretation of complex requirements before proceeding
- **Request Examples**: Ask for specific use cases when general requests are too broad

### Quality Checks
- No duplicate information across sections
- Technical accuracy over simplification
- Focus on "why" and "how" rather than just "what"
- Include relevant Spring Security version considerations when applicable

## Clarification Questions to Ask
- What specific aspect needs deeper explanation?
- Are you looking for production-ready patterns or learning examples?
- Do you need database integration alternatives to in-memory examples?
- Should I cover security implications and best practices?
- Are there specific integration scenarios (OAuth, LDAP, custom providers) to address?