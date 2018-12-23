answer your own questions

lots of crap on the internet about answering questions. this isn't prescriptive.

had a q: how to get workspace name in bazel for template expansion

started writing email

realised that the python rules, while native, must do the thing

built a hello world binary, searched for workspace name in output

searched for the template stub file in source (rg PYTHON_BINARY -l)

searched for where it was plugged in to get template variable

searched for workspace_name

found it uses getworksspacename on rulecontext

searched skylarkrulecontext for same

found workspace_name, which was in docs all along

I'd been looking for repository name
