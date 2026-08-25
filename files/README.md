# files/

This directory is mounted into the n8n container (see `docker-compose.yml`,
`./files:/home/node/.n8n-files`) and is where the workflow's `Load Resume File`
node reads the attachment from.

Drop your resume here as `resume.pdf` (or update `resume_file_name` /
`resume_file_path` in the **Workflow Config** node to match a different name).

The actual PDF is git-ignored (see `.gitignore`) so your resume never gets
committed — this README is the only thing in this folder that's tracked, just
so the directory itself shows up when someone clones the repo.
