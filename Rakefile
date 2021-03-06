require 'pathname'
require 'cocoapods-core'
require 'cocoapods'

# Configuration
#-----------------------------------------------------------------------------#

Pod::Config.instance.repos_dir = Pathname.pwd.dirname
Pod::Config.instance.verbose = true

PODS_ALLOWED_TO_FAIL = [
  'PinEntry',
  'LibComponentLogging-pods',
]


#-----------------------------------------------------------------------------#

# TODO pass old spec
# TODO catch spec eval raise
desc "Run `pod spec lint` on all specs"
task :validate do
  exit if ENV['skip-lint']

  title('Most Recently Commited Specs ')
  puts "The Master repo will not accept specifications with warnings."
  puts "The specifications from the most recent commit are linted with the most strict settings."
  puts "For more information see: http://docs.cocoapods.org/guides/contributing_to_the_master_repo.html"

  has_commit_failures = false
  last_commit_specs.each do |spec_path|
    puts "\n#{spec_path}"
    spec = Pod::Spec.from_file(spec_path)
    acceptable = check_if_can_be_accepted(spec, spec_path)
    if ENV['TRAVIS_PULL_REQUEST'] && ENV['TRAVIS_PULL_REQUEST'] != 'false'
      lints = lint(spec)
    else
      lints = quick_lint(spec)
    end

    if acceptable && lints
      puts green("- The spec can be accepted.")
    else
      has_commit_failures = true
    end
  end

  report = generate_health_report
  puts "\n\n\n"
  print_health_report(report)

  if has_commit_failures || !report_acceptable(report)
    puts red("Validation failed!")
    exit 1
  else
    puts green("Validation passed")
  end
end

#-----------------------------------------------------------------------------#

desc "Checks the repo for errors or warnings"
task :health_report do
  report = generate_health_report
  puts "\n\n\n"
  print_health_report(report)
end

#-----------------------------------------------------------------------------#

desc "Deprecated task which was used by git pre-commit hook"
task :lint do
  puts
  puts yellow("The pre-commit hook of the master repo has been deprecated.")
  puts "You can remove it by running:"
  puts
  puts "    $ rm -i ~/.cocoapods/master/.git/hooks/pre-commit"
  puts
  puts "Please lint you specifications manually before submitting the to the"
  puts "specs repo. To do so you can either use:"
  puts
  puts "    $ pod push [ REPO ] [NAME.podspec]"
  puts "    $ pod spec lint [ NAME.podspec | DIRECTORY | http://PATH/NAME.podspec, ... ]"
end

task :default => :validate

# group Analysis helpers
#-----------------------------------------------------------------------------#

# @return [Bool] If the spec can be accepted
#
def check_if_can_be_accepted(spec, spec_path)
  # previous_spec_contents = previous_version_of_spec(spec_path)
  acceptor = Pod::Source::Acceptor.new('.')
  errors = acceptor.analyze(spec)
  errors.each do |error|
    puts red("- #{error}")
  end
  errors.count.zero?
end

# @return [Bool] Whether the spec lints
#
def lint(spec)
  validator = Pod::Validator.new(spec)
  validator.validate
end

# @return [Bool] Whether the spec lints
#
def quick_lint(spec)
  linter = Pod::Spec::Linter.new(spec)
  linter.lint
  linter.results.each do |result|
    puts red("- #{result}")
  end
  linter.results.count.zero?
end

# @return [Pod::Source::HealthReport] Returns the health report of the repo.
#
def generate_health_report
  title('Health Report')
  reporter = Pod::Source::HealthReporter.new('.')
  reporter.master_repo_mode = true
  count = 0
  reporter.pre_check do |name, version|
    count += 1
    if (count % 20) == 0
      print '.'
    end
  end
  reporter.analyze
end

def report_acceptable(report)
  report.pods_by_error.values.all? do |pod_info|
    pod_info.keys.all? do |pod_name|
      PODS_ALLOWED_TO_FAIL.include?(pod_name)
    end
  end
end

# group Git helpers
#-----------------------------------------------------------------------------#

# @return [Array<String>] Returns the relative path of the podspecs affected by
#         the last commit.
#
def last_commit_specs
  specs = `git diff-index --name-only HEAD | grep '.podspec$'`.strip.split("\n")
  specs = ['.'] if specs.empty?
  `git diff --diff-filter=ACMRTUXB --name-only HEAD~1..HEAD | grep '.podspec$'`.strip.split("\n")
end

# @return [String] The contents of the given specification before the last
#         commit.
#
def previous_version_of_spec(spec_path)
  `git show HEAD~1:#{spec_path}`
end

# group UI helpers
#-----------------------------------------------------------------------------#

# Prints a title.
#
def title(title)
  cyan_title = "\033[0;36m#{title}\033[0m"
  puts
  puts "-" * 80
  puts cyan_title
  puts "-" * 80
  puts
end

# Colorizes a string to green.
#
def green(string)
  "\033[0;32m#{string}\e[0m"
end

# Colorizes a string to yellow.
#
def yellow(string)
  "\033[0;33m#{string}\e[0m"
end

# Colorizes a string to red.
#
def red(string)
  "\033[0;31m#{string}\e[0m"
end

# @return [void] Prints the given health report.
#
def print_health_report(report)
  report.pods_by_error.keys.sort.each do |message|
    versions_by_name = report.pods_by_error[message]
    puts red("-> #{message}")
    versions_by_name.each { |name, versions| puts "  - #{name} (#{versions * ', '})" }
    puts
  end

  report.pods_by_warning.keys.sort.each do |message|
    versions_by_name = report.pods_by_warning[message]
    puts yellow("-> #{message}")
    versions_by_name.each { |name, versions| puts "  - #{name} (#{versions * ', '})" }
    puts
  end

  puts "Analyzed #{report.analyzed_paths.count} podspecs files."
end

#-----------------------------------------------------------------------------#

module Pod
  # Suppress the warnings because they make too much noise at this stage.
  def CoreUI.warn(message)
  end
end


