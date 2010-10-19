require 'yaml'
require 'pp'
require 'ruby-debug'
require 'pathname'
require 'popen4'

CONFIG = OpenStruct.new(YAML::load_file('config.yml'))
VERBOSE = ENV['VERBOSE'] || false

task :default => [:run]

desc 'Compare Tack output and normal test output'
task :run do
  results = {:success => [], :failure => [], :skipped => []}
  if ENV['PROJECT']
    run_and_compare(ENV['PROJECT'], results)
  else
    if CONFIG.projects
      CONFIG.projects.each do |dir|
        run_and_compare(CONFIG.projects_dir+dir, results)
      end
    else
      # run all projects under projects_dir
      Pathname.new(CONFIG.projects_dir).children.each do |file|
        next if !file.directory?
        run_and_compare(file, results)
      end
    end
  end
  puts "=== Summary ==="
  puts "#{results[:success].length} succeeded, #{results[:failure].length} failed, #{results[:skipped].length} skipped"
  results[:failure].each do |result|
    puts "FAILED: #{result}"
  end
  results[:skipped].each do |result|
    puts "SKIPPED: #{result}"
  end

end

def check_adapter(config, results, project_dir)
  if config.adapters
    mapping = config.adapters
    command = "#{CONFIG.tack_command} --adapters"
    status, output = run_command_in_rvm(command)
    if status != 0 
      puts "Running command '#{command}' failed: "
      puts output
      results[:success].delete(project_dir)
      results[:failure] << project_dir
    else
      mapping.each do |path, adapter|
        pattern = %r{directory .*/?#{path} use .*#{adapter}}
        unless output=~pattern
          puts "--> The actual adapter did not match the expected adapter"
          puts "\n\n#{output}\n\n does not match #{pattern})"
          results[:success].delete(project_dir)
          results[:failure] << project_dir
        end
      end
    end
  end
end

def run_and_compare(project_dir, results)
  puts "--> Checking #{project_dir} ... "
  Dir.chdir(project_dir) do
    config_file = 'tack-test.yml'
    config_hash = File.exists?(config_file) ? YAML::load_file(config_file) : {}
    local_config = OpenStruct.new(config_hash)
    if local_config.skip
      puts "--> Skipping #{project_dir}"
      puts "--> (#{local_config.reason})" if local_config.reason
      results[:skipped] << project_dir
      return
    else
      status, test_output, _ = run_tests(local_config)
      status, tack_output, _ = run_tack(CONFIG)
      if !same_output?(test_output, tack_output)
        results[:failure] << project_dir
      else
        results[:success] << project_dir
      end
      check_adapter(local_config, results, project_dir)
    end
  end
end

def same_output?(expected_output, tack_output)
  same = true
  match = tack_output.match(/(\d+) tests, (\d+) failures, (\d+) pending/)
  return false if match.nil?
  tack_tests, tack_failures, tack_pending = match[1..3].map(&:to_i)
  
  # TODO - refactor to be more clear depending on the test framework
  match = expected_output.match(/(\d+) tests, \d+ assertions, (\d+) failures, (\d+) errors/)
  if !match.nil? # It's Test::Unit
    expected_pending = expected_output.scan(/DEFERRED\:/).length
    expected_tests = match[1].to_i + expected_pending
    expected_failures = match[2].to_i + match[3].to_i
  else # It's not Test::Unit
    # 28 examples, 15 failures, 3 pending
    match = expected_output.match(/(\d+) examples, (\d+) failures, (\d+) pending/)
    if !match.nil? 
      expected_pending = match[3].to_i
    else # No pending tests
      expected_pending = 0
      match = expected_output.match(/(\d+) examples, (\d+) failures/)
    end
    expected_tests = match[1].to_i
    expected_failures = match[2].to_i
  end
  
  if tack_tests != expected_tests
    same = false
    puts "!!! Tack reported #{tack_tests} tests while Ruby reported #{expected_tests} !!!" if VERBOSE
  end
  if tack_failures != expected_failures
    same = false
    puts "!!! Tack reported #{tack_failures} failures while Ruby reported #{expected_failures} !!!" if VERBOSE
  end
  if tack_pending != expected_pending
    same = false
    puts "!!! Tack reported #{tack_pending} pending while Ruby reported #{expected_pending} !!!" if VERBOSE
  end
  same
end

def run_tack(config)
  command = config.tack_command
  run_command_in_rvm(command)
end

def run_tests(config)
  command = config.test_command || 'rake'
  run_command_in_rvm(command)
end

def run_command_in_rvm(command)
  rvm_command = "source $HOME/.rvm/scripts/rvm;"
  if File.exists?('.rvmrc')
    rvm_command += "source .rvmrc;"
  else
    rvm_command += "rvm use 1.8.7;"
  end
  rvm_command += command
  bash_command = "bash -c '#{rvm_command}'"
  run_command(bash_command, VERBOSE)
end

def run_command(command, echo = false)
  puts "Running : #{command}" if echo 
  stdout = stderr = nil
  status = POpen4::popen4(command) do |out, err, _, _|
    stdout = out.read
    stderr = err.read
  end
  exit_status = status.exitstatus
  if exit_status != 0
    if echo
      puts "Command exited with status #{exit_status}"
      puts "STDOUT: "
      puts stdout
      puts "STDERR: "
      puts stderr
    end
  end
  [exit_status, stdout, stderr]
end

puts "Done"



