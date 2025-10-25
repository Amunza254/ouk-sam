# ouk-sam
GAT 
sam amunza 
/*
==========================================================
 Codebase Genius  –  Autonomous Code-Documentation System
==========================================================
Author:        <Your Name>
Repository:    https://github.com/<your-github-username>/agentic_codebase_genius
Language:      Jac (JacLang)
----------------------------------------------------------
Description:
A multi-agent Jac system that generates high-quality
Markdown documentation for any public GitHub repository.

Agents (Walkers):
  1. code_genius        – Supervisor / API entrypoint
  2. repo_mapper        – Builds file-tree map of the repo
  3. readme_summarizer  – Summarises README.md
  4. code_analyzer      – Builds Code Context Graph (CCG)
  5. doc_genie          – Synthesises final documentation

Run:
  jac serve main.jac

Then POST a request to:
  http://localhost:8000/walker_run
  {
    "name": "code_genius",
    "ctx": {"repo_url": "https://github.com/psf/requests"}
  }
==========================================================
*/

// ---------------------------------------------------------
// NODE DEFINITIONS
// ---------------------------------------------------------

node file_tree {
    has structure;          // Nested dict of repo file structure
}

node readme_summary {
    has content;            // LLM-based summary of README.md
}

node function_node {
    has name;               // Function or class name
    has file;               // File where it’s defined
}

node doc_fragment {
    has markdown;           // Partial Markdown block
}

// ---------------------------------------------------------
// WALKER: Repo Mapper
// ---------------------------------------------------------
walker repo_mapper {
    has repo_path;                                  // Local cloned repo path
    has ignore_dirs = [".git","__pycache__","node_modules",".venv","dist","build"];
    has tree_representation;

    can visit, spawn;

    init {
        report("📁 [RepoMapper] Mapping repository: " + repo_path);
        if !file::exists(repo_path) {
            report("❌ Repo path not found: " + repo_path);
            return;
        }
        tree_representation = self::scan_dir(repo_path);
        spawn node::file_tree(structure=tree_representation);
        report("✅ [RepoMapper] Repository mapping complete.");
    }

    can self::scan_dir(path) {
        // Recursively build a directory tree representation
        entries = [];
        for entry in file::list_dir(path) {
            if entry in ignore_dirs { continue; }
            full_path = path + "/" + entry;
            if file::is_dir(full_path) {
                child = self::scan_dir(full_path);
                entries += [{"type":"dir","name":entry,"children":child}];
            } else {
                entries += [{"type":"file","name":entry}];
            }
        }
        return entries;
    }
}

// ---------------------------------------------------------
// WALKER: README Summariser
// ---------------------------------------------------------
walker readme_summarizer {
    has repo_path;

    init {
        report("🧾 [ReadmeSummariser] Searching for README...");
        readme_file = repo_path + "/README.md";
        if !file::exists(readme_file) {
            report("⚠️ No README.md found, skipping summary.");
            return;
        }
        content = file::read_text(readme_file);

        // --- Replace the line below with your chosen LLM provider ---
        summary = byllm::prompt(
            "Summarise this project README in 5 concise sentences:\n" + content
        );
        // ------------------------------------------------------------

        spawn node::readme_summary(content=summary);
        report("✅ [ReadmeSummariser] Summary node created.");
    }
}

// ---------------------------------------------------------
// WALKER: Code Analyzer (Tree-sitter + CCG builder)
// ---------------------------------------------------------
walker code_analyzer {
    has repo_path;
    has language = "python";

    init {
        report("🔍 [CodeAnalyzer] Building Code Context Graph...");
        if !file::exists(repo_path) { report("❌ Invalid repo path."); return; }

        files = self::collect_source_files(repo_path);
        for f in files { self::analyze_file(f); }
        report("✅ [CodeAnalyzer] Analysis complete. Graph nodes created.");
    }

    can self::collect_source_files(path) {
        // Collect .py and .jac files recursively
        result = [];
        for entry in file::list_dir(path) {
            full = path + "/" + entry;
            if file::is_dir(full) {
                result += self::collect_source_files(full);
            } else if file::ends_with(entry,".py") or file::ends_with(entry,".jac") {
                result += [full];
            }
        }
        return result;
    }

    can self::analyze_file(file_path) {
        try {
            code = file::read_text(file_path);
            // --- Example call to external Python Tree-sitter module ---
            // parsed = parser_module::analyze_code(code, language);
            // for func in parsed.functions { spawn node::function_node(name=func.name, file=file_path); }
            // for rel in parsed.calls { connect node::function_node(rel.src) -> node::function_node(rel.dst); }
            // ----------------------------------------------------------------
            // Placeholder simple analysis:
            if string::contains(code,"def ") {
                spawn node::function_node(name="functions_in_" + file::basename(file_path), file=file_path);
            }
        } catch {
            report("⚠️ Error analyzing " + file_path);
        }
    }
}

// ---------------------------------------------------------
// WALKER: Doc Genie – Markdown Assembler
// ---------------------------------------------------------
walker doc_genie {
    has repo_name;
    has output_dir = "./outputs/";

    init {
        report("🪶 [DocGenie] Synthesising documentation...");
        summary_nodes = get_nodes(node::readme_summary);
        file_nodes = get_nodes(node::file_tree);
        function_nodes = get_nodes(node::function_node);

        markdown = "# " + repo_name + " – Codebase Documentation\n\n";

        if len(summary_nodes) > 0 {
            markdown += "## Overview\n" + summary_nodes[0].content + "\n\n";
        }

        if len(file_nodes) > 0 {
            markdown += "## File Structure\n";
            markdown += format::to_markdown_tree(file_nodes[0].structure) + "\n\n";
        }

        markdown += "## API Reference\n";
        for f in function_nodes {
            markdown += "- `" + f.name + "`  (" + f.file + ")\n";
        }

        output_path = output_dir + repo_name + "/docs.md";
        file::make_dirs(output_dir + repo_name);
        file::save_text(output_path, markdown);
        report("✅ [DocGenie] Documentation saved to " + output_path);
    }
}

// ---------------------------------------------------------
// WALKER: Code Genius (Supervisor + API entrypoint)
// ---------------------------------------------------------
walker code_genius {
    has repo_url;
    has repo_path;
    has repo_name;
    has output_path;

    init {
        report("🚀 [CodeGenius] Starting pipeline...");

        if !repo_url or string::len(repo_url)==0 {
            report("❌ Missing repo_url in request context."); return;
        }

        repo_name = string::split(repo_url,"/")[-1];
        repo_path = "./tmp/" + repo_name;

        try {
            report("⬇️  Cloning " + repo_url);
            os::system("git clone " + repo_url + " " + repo_path);
        } catch {
            report("❌ Failed to clone repository. Check URL or access.");
            return;
        }

        spawn -> repo_mapper(repo_path=repo_path);
        spawn -> readme_summarizer(repo_path=repo_path);
        spawn -> code_analyzer(repo_path=repo_path);
        spawn -> doc_genie(repo_name=repo_name);

        output_path = "./outputs/" + repo_name + "/docs.md";
        report("✅ [CodeGenius] Documentation pipeline finished. Output → " + output_path);
    }
}
