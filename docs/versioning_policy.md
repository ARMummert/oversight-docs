# Oversight ICS Versioning Policy

**Version**: 1.0.0 

## API Versioning & Deprecation Policy

1. **Version Identification**: All breaking changes to the schema (e.g., changing quality from an integer to an object)
   trigger a major version bump (v1 $\rightarrow$ v2).
2. **N-1 Support Requirement**: The Oversight ICS Edge Gateway must support the current version ($N$) and the 
   immediate predecessor ($N$-1) simultaneously.
3. **Deprecation Policy**:
   - **Notice Period**: 3 months before a new version is released.
   - **Overlap Period**: 12 months when both $N$ and $N-1$ are supported.
   - **Sunset**: $N$-1 is only decommissioned after 12 months and a verified **NO-TRAFFIC** audit from the IDMZ logs.
4. **Emergency Patches**: Security patches (JWT logic fixes) are applied to all active versions immediately and do not
   trigger a version increment.