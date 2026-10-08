# VulnBank-Demo fixture

A minimal, synthetic Android manifest written specifically for the "Steps to Reproduce" walkthrough in [`../steps-to-reproduce.md`](../steps-to-reproduce.md). It is not a real app, not downloaded from anywhere, and not based on any specific published application. It exists purely so the walkthrough can run the skill's real agents against real content and show genuine tool output, without needing to download a third-party APK.

It intentionally contains a small number of well-known manifest misconfigurations (debuggable build, unrestricted backup, an exported activity with a sensitive-sounding name and no permission) so the pipeline has something real to find.
