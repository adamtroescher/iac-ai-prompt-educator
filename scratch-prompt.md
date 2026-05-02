# Goal

Hello! Using python. I would like to create a terraform infrastructure as code prompt engineering training program. The tool should act like an educator that evaluates the responses back from claude and grades them against a base set of criteria. Supplemental criteria can also be added by the user as markdown files.

## Audience

The target audience is developers and data scientists that are new to terraform and infrastructure as code.

## Token Usage

This tool should be written in a way that is very conscious of the amount of tokens it consumes. Storing text or context in a local directory that is gitignored is an okay workaround for keeping the context low to save on tokens. 

## Score

Should be on a 0 being the worst to 100 being the best.

Score the prompt from 0 to 100 based on these criteria:

1. **Specificity** (0–25): Does the prompt name the cloud provider, resource types, region, environment (dev/staging/prod), or other concrete details? Vague requests score low.

2. **Context & Intent** (0–25): Does the prompt explain *why* the infrastructure is needed, what it connects to, or what problem it solves? Context helps the model avoid wrong assumptions.

3. **Constraints & Requirements** (0–25): Are security, compliance, cost, naming conventions, tagging, or other guardrails mentioned? Missing constraints lead to generic or unsafe output.

4. **Completeness** (0–25): Are all the inputs needed to generate a working module present? Missing values like instance sizes, CIDR blocks, or variable names force guessing.

## Usage

This is a local only python utility or program that users will clone locally then run through some sort of startup process which also bootstraps the tool.

## Bootstrap

The user should be prompted for a claude api key and it should be stored locally in the repo in a .env file which is excluded from source control. A warning about consuming claude tokens at their own expense should be shown and highlighted.

## Base criteria

Infrastructre as code best practices should be followed. Loops should be used when possible to avoid copy and pasting. Cost should be considered and RBAC using least privledge should be accounted for.

## Interaction

I would like the user interface to be a terminal-based (CLI) user interface. It is a keyboard-driven, text-based interface designed to run directly inside your terminal emulator. K9s is an example of this user interface. The UI should show the last prompt the user created including the score and some suggestions about prompt improvemnt. The UI should also include a place for the user to type the prompt.

## Additional Evaluation Context

Users should be able to add in their own additional evaluation content. This should take the shape of a markdown document. I have created an example but feel free to update the structure of the markdown document to be modular.

<example_additional_evaluation_content>
# Azure Cloud Provider
These are for azure resources created in a secure enterprise tenant.
## Required
- A private registry at registry.terraform.io/movingpictures/ must be used
- Publicly accessible resources must be secured with at least a firewall
## Optional
- Use private endpoints and networking where possible
- Use for loops where possible to avoid copy and paste
</example_additional_evaluation_content>

## Example interactions

<prompt 1>
Prompt: do an azure storage account
Score: low
Suggestions: What type of azure storage account? LRS or GRS?
</prompt 1>

<prompt 2>
Prompt: create three azure storage accounts, no global replication is needed, define all three in a locals block for an iterator
Score: medium
Suggestions: What type of azure storage account? LRS or GRS?
</prompt 2>

<prompt 3>
Prompt: Create three azure storage accounts. No global replication is needed so use LRS. Define all three in a locals block for an iterator. Specify reasonable default variables for the inputs and only override what is needed through the locals block. When a user specified value is expected add checks and conditionals to verify the input matches the naming standard that starts with azstor
Score: high
Suggestions: What type of azure storage account? LRS or GRS?
</prompt 3>