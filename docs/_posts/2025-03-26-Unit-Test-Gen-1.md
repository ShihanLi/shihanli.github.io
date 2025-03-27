---
layout: post
title:  "Automated Unit Test Generation Pt.1"
date:   2025-03-26 23:24:43 -0800
categories: projects
---
# Automated Unit Test Generation Pt.1

<h2>The Opportunity</h2>

Unit testing follows standardized patterns with significant boilerplate code. Across our codebases, similar components (controllers, message handlers, business logic classes) share similar testing scenarios. This predictable structure makes unit testing generation an ideal candidate for leveraging large language models.

Even more important, comprehensive testing serves as a safety net to validate that code behaves as expected in the age of AI-assisted development. It's makes us more confident to deploy other AI tools into the development workflow.

<h2>Pain Points with Copilot</h2>

Our team has traditionally used Copilot for test generation, but we've encountered several inefficiencies:
1. **Inconsistent Results**: Everyone has their own prompts. The generated tests vary significantly in style and accuracy
2. **Repetitive Manual Work**: Each person has to repeatedly copy prompts, upload files, iterate on model outputs each time they ask copilot to generate tests.
3. **Slow Knowledge Sharing**: If one person improves the prompt to generate more accurate tests, the friction to sharing that prompt and everyone else adopting it is high.

<h2>Our Project</h2>
We've developed a Visual Studio extension that streamlines the entire test generation process. With just a right click on any code file, our tool automatically generates standardized, accurate unit tests and copies them directly to the clipboard - ready to paste into a new test file.

Our solution addresses each of the pain points:
1. The extension enforces the same styling that's consistent with the rest of the repo through stylecop
2. It replaces the manual workflow with a single click.
3. Each iteration or optimization on our prompt automatically benefits all users. 

<h3>Steps Taken</h3>
Our solution evolved through these 5 stages:

1. **Basic Prompt Engineering**: We stood up a visual studio extension with only the system prompt: "You are a C# developer specialized in unit tests" and user prompt: "Generate unit test for {{CODE}}" on day one. This allowed us to iterate fast on the prompt, models, and evaluations we used to improve the accuracy of the tests generated.
2. **Model Fine-Tuning**: We wrote a PowerShell script to create a training data set consisting of code files - test files pairs, which we fed to Azure AI Foundry to fine tune a GPT-4o model. Surprisingly, the tests generated have more build errors and more stylistically inconsistent with our team's code. We suspect it's because of potential biases in our training data or insufficient training examples.
3. **Style Guide Integration**: We found the model generates the best code when we directly inject our entire style cop file to the system prompt. 
4. **Hardcoded Examples**: We attempted to provide specific examples of tests that we thought was well written in the user prompt. This improved the accuracy of the generated tests slightly, but not as much as we liked.
5. **Dynamic Examples**: We enabled the developers to select a reference example that's functionally closest to their test class. This introduced a manual step in the flow, but significantly improved the quality of the tests that we decided to keep it. 


<h3>Challenges</h3>
Our primary challenge has been integration issues between the legacy Visual Studio extension template and the newer Azure.AI.Inference package. Currently, our project runs with multiple package version conflicts, which has prevented us from merging this tool into Intune's repositories.

<h2>Plans for the Future</h2>

1. **Automate Example Selection**: We know that asking developers to manually identify relevant examples is a significant friction point. We plan on indexing our codebases using semantic embeddings to automatically identify functionally similar components. This means controllers handling REST endpoints will cluster with other controllers, while remaining distinct from data stores. This way, we can automatically pick the most relevant examples.
2. **Dynamic Context Injection**: We found that the LLM sometimes hallucinates because it's missing relevant interface and type definitions that are referenced in the system under test. Using Roslyn, we can automatically identify these dependencies and inject them into the context.
3. **Robust Eval System**: Unlike open ended text generation, code quality offers concrete metrics for evaluation. We hope to create an automated pipeline to track build errors, warnings, and test pass rates across generations. 

<h3>Core Logic (some parts are cut off due to hackbox limitations)</h3>

```csharp
        private void Execute(object sender, EventArgs e)
        {
            ThreadHelper.ThrowIfNotOnUIThread();

            var dte = Package.GetGlobalService(typeof(DTE)) as DTE;
            if (dte == null)
            {
                Debug.WriteLine("DTE is null");
                return;
            }

            var progressDialog = Package.GetGlobalService(typeof(SVsThreadedWaitDialog)) as IVsThreadedWaitDialog2;
            if (progressDialog == null)
            {
                Debug.WriteLine("Progress dialog is null");
                return;
            }

            var exampleFilePath = this.PrompUserForFile(dte, "Please select an existing file with similar functionalities that has unit tests");
            if (string.IsNullOrEmpty(exampleFilePath))
            {
                return;
            }
            var exampleFileContent = File.ReadAllText(exampleFilePath);

            var exampleTestFilePath = this.PrompUserForFile(dte, "Please select the corresponding unit test for that file");
            if (string.IsNullOrEmpty(exampleTestFilePath))
            {
                return;
            }
            var exampleTestFileContent = File.ReadAllText(exampleTestFilePath);

            progressDialog.StartWaitDialog(
                "Generating unit tests",
                "Reading source file",
                null,
                null,
                null,
                0,
                true,
                true);

            var systemPrompt = @$"You are an expert C# developer specializing in comprehensive unit testing. You task is to generate production-ready unit tests for C# class following Microsoft's enterprise coding standards.

Instructions:
Use xUnit as the testing framework
Follow Arrange-Act-Assert pattern (explicit comments not needed)
Use the Moq library for mocking dependencies
Use the FluentAssertions library for Assert statements
Name test methods using the convention ""MethodName_Scenario_ExpectedResult""
Test both happy path and edge cases
Leverage the AutoMoqData custom attribute, which automatically generate mock data for test methods that's specified in the parameters

The tests you write must be consistent with our StyleCop configuration: {StyleCop}, and specifically
1. parameters should begin on the line after the declaration, whenever the parameter spans across multiple lines
2. lines should not contain trailing whitespaces
3. the parameters should all be placed on the same line or each parameter should be placed in its own line
4. use trailing comma in multiline initializers

The generated code must be ready for direct use without modification. Your output must contain only valid C# code with no explanatory text, comments outside the code, or formatting errors.";

            var filePath = dte.ActiveDocument.FullName;
            var fileContent = File.ReadAllText(filePath);

            progressDialog.UpdateProgress(
                "Preparing API Reqeust",
                "Read in active document",
                null,
                1,
                2,
                true,
                out var cancelled);

            if (cancelled)
            {
                return;
            }

            var endpoint = new Uri(AzureOpenAIEndpoint);
            var credential = new AzureKeyCredential(AzureOpenAIKey);

            var client = new ChatCompletionsClient(endpoint, credential, new ChatCompletionsClientOptions());
            var requestOptions = new ChatCompletionsOptions()
            {
                Messages =
                {
                    new ChatRequestSystemMessage(systemPrompt),
                    new ChatRequestUserMessage($"Here's an example of a good test. The class: {exampleFileContent}, the tests: {exampleTestFileContent} Please write unit tests for: {fileContent}"),
                },
                Model = AzureOpenAIModel,
            };

            progressDialog.UpdateProgress(
                "Contacting Azure OpenAI",
                "Sending request to generate tests, this might take a minute...",
                null,
                2,
                2,
                true,
                out cancelled);

            if (cancelled)
            {
                return;
            }

            var stopwatch = Stopwatch.StartNew();
            this.telemetry.TrackEvent("UnitTestGenerationStarted", new Dictionary<string, string>
            {
                { "FilePath", filePath },
                { "FileContentLength", fileContent?.Length.ToString() ?? "0" }
            });

            var response = client.Complete(requestOptions);
            var generatedTests = response.Value.Choices[0].Message.Content;

            stopwatch.Stop();
            progressDialog.EndWaitDialog();
            this.telemetry.TrackEvent("UnitTestGenerationSucceeded", new Dictionary<string, string>
            {
                { "ResponseTime", stopwatch.ElapsedMilliseconds.ToString() },
                { "GeneratedTestsLength", generatedTests?.Length.ToString() ?? "0" }
            });

            if (!string.IsNullOrEmpty(generatedTests))
            {
                Clipboard.SetText(generatedTests);
                MessageBox.Show("Unit tests copied to clipboard!", "Success", MessageBoxButtons.OK);
            }
            else
            {
                MessageBox.Show("Failed to generate unit tests", "Error", MessageBoxButtons.OK);
            }
        }
```