git_with_build_tools
====================

Example of the automation of the `git` tasks with `build_tools`.

Version: 0.0.7

You can download this project and play with it.

The build script located at `tool/project.dart`.

```dart
import "dart:io";
import "package:build_tools/build_shell.dart";
import "package:build_tools/build_tools.dart";
import "package:build_tools/build_utils.dart";
import "package:file_utils/file_utils.dart";

const String CHANGE_LOG = "change.log";
const String CHANGELOG_MD = "CHANGELOG.md";
const String README_MD = "README.md";
const String README_MD_IN = "README.md.in";
const String PUBSPEC_YAML = "pubspec.yaml";

void main(List<String> args) {
  // http://dartbug.com/20119 (before change directory)
  var script = Platform.script;

  // Change directory to root
  FileUtils.chdir("..");

  // For use in `README.md`
  var thisFile = script.toFilePath();

  file(CHANGELOG_MD, [CHANGE_LOG], (Target t, Map args) {
    writeChangelogMd();
  });

  file(README_MD_IN, [], (Target t, Map args) {
    FileUtils.touch([t.name], create: true);
  });

  file(README_MD, [README_MD_IN, thisFile], (Target t, Map args) {
    var template = new File(t.sources.first).readAsStringSync();
    var example = new File(t.sources.last).readAsStringSync();
    template = template.replaceFirst("{{EXAMPLE}}", example);
    template = template.replaceFirst("{{VERSION}}", getVersion());
    new File(t.name).writeAsStringSync(template);
  });

  target("default", ["git:status"], null, description: "git status");

  target("git:status", [], (Target t, Map args) {
    return exec("git", ["status", "--short"]);
  }, description: "git status --short");

  target("git:add", [], (Target t, Map args) {
    return exec("git", ["add", "--all"]);
  }, description: "git add --all");

  target("git:commit", [CHANGELOG_MD, README_MD, "git:add"], (Target t, Map
      args) {
    var message = args["m"];
    if (message == null || message.isEmpty) {
      print("Please, specify the `commit` message with --m option");
      return -1;
    }

    return exec("git", ["commit", "-m", message]).then((exitCode) {
      if (exitCode == 0) {
        updateVersion(incrementVersion(getVersion()));
        print("Version switched to ${getVersion()}");
      }

      return exitCode;
    });
  }, description: "git commit, --m \"message\"");

  target("git:push", [], (Target t, Map args) {
    // TODO: The `exec git push` does not show the prompt on Windows.
    // But on Posix the `exec git push` works as expected.
    return exec("git", ["push", "origin", "master"]);
  }, description: "git push origin master");

  target("log:changes", [], (Target t, Map args) {
    var message = args["m"];
    if (message == null || message.isEmpty) {
      print("Please, specify the `message` with --m option");
      return -1;
    }

    logChanges(message);
  }, description: "log changes, --m message", reusable: true);

  target("prj:changelog", [CHANGELOG_MD], null, description:
      "generate '$CHANGELOG_MD'", reusable: true);

  target("prj:version", [], (Target t, Map args) {
    print("Version: ${getVersion()}");
  }, description: "display version", reusable: true);

  new BuildShell().run(args).then((exitCode) => exit(exitCode));
}

String getVersion() {
  var file = new File(PUBSPEC_YAML);
  var lines = file.readAsLinesSync();
  var version = "0.0.1";
  for (var line in lines) {
    if (line.startsWith("version")) {
      var index = line.indexOf(":");
      if (index != -1 && line.length > index + 1) {
        version = line.substring(index + 1).trim();
      }
    }
  }

  return version;
}

String incrementVersion(String version) {
  var parts = version.split(".");
  if (parts.length < 3) {
    return version;
  }

  var patch = int.parse(parts[2], onError: (x) => null);
  if (patch == null) {
    return version;
  }

  parts[2] = ++patch;
  parts.length = 3;
  return parts.join(".");
}

void logChanges(String message) {
  if (message == null || message.isEmpty) {
    return;
  }

  FileUtils.touch([CHANGE_LOG], create: true);
  var file = new File(CHANGE_LOG);
  var length = file.lengthSync();
  var fp = file.openSync(mode: FileMode.APPEND);
  var sb = new StringBuffer();
  if (length != 0) {
    sb.writeln();
  }

  sb.write("${getVersion()} $message");
  fp.writeStringSync(sb.toString());
  fp.closeSync();
}

void updateVersion(String version) {
  var file = new File(PUBSPEC_YAML);
  var found = false;
  var lines = file.readAsLinesSync();
  var sb = new StringBuffer();
  for (var line in lines) {
    if (line.startsWith("version")) {
      found = true;
      break;
    }
  }

  if (!found) {
    var pos = lines.length == 0 ? 0 : 1;
    lines.insert(pos, "version: $version");
  }

  for (var line in lines) {
    if (line.startsWith("version")) {
      sb.writeln("version: $version");
    } else {
      sb.writeln(line);
    }
  }

  var string = sb.toString();
  file.writeAsStringSync(string);
}

void writeChangelogMd() {
  FileUtils.touch([CHANGELOG_MD], create: true);
  var log = new File(CHANGE_LOG);
  if (!log.existsSync()) {
    return;
  }

  var lines = log.readAsLinesSync();
  lines = lines.reversed.toList();
  var versions = <String, List<String>> {};
  for (var line in lines) {
    var index = line.indexOf(" ");
    if (index != -1) {
      var version = line.substring(0, index);
      var message = line.substring(index + 1).trimLeft();
      var messages = versions[version];
      if (messages == null) {
        messages = <String>[];
        versions[version] = messages;
      }

      messages.add(message);
    }
  }

  var sb = new StringBuffer();
  for (var version in versions.keys) {
    sb.writeln("**${version}**");
    sb.writeln();
    var messages = versions[version];
    messages.sort((a, b) => a.compareTo(b));
    for (var message in messages) {
      sb.writeln("- $message");
    }

    sb.writeln();
  }

  var md = new File(CHANGELOG_MD);
  md.writeAsStringSync(sb.toString());
}

```
