# In-cluster PKI

This kind of deployment is generally not advisable, for several reasons:

1. Without a lot of RBAC and policy, this isn't particularly safe!
2. Just appying YAML will work, but doesn't come with a plan for rotation!
3. Difficult to do this safely in a multi-cluster environment
