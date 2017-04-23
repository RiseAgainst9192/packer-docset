require 'rake'
require 'sequel'
require 'nokogiri'
require 'logger'
require 'pry'

URL = 'www.packer.io'.freeze

task default: [:local_build]

desc 'Build the docset'
task build: %i[clean download local_build]

task :clean do
  system('rm -rf hts-* backblue.gif fade.gif index.html cookies.txt')
  system('rm -f docSet.dsidx')
  system("rm -rf #{URL}")
end

task :local_build do
  system('rm -rf Packer.docset')
  system('rm -f docSet.dsidx')

  Rake::Task['remove_non_needed_html'].execute
  Rake::Task['make_database'].execute
  Rake::Task['build_docset'].execute
  Rake::Task['compress_docset'].execute
end

task :download do
  puts 'Downloading documentation...'
  system("httrack http://#{URL}/docs --mirror")
  system("httrack http://#{URL}/docs --update")
end

task :remove_non_needed_html do
  Dir.glob("#{URL}/**/*.html").each do |filename|
    doc = Nokogiri::HTML(File.open(filename))

    doc.search('//div[contains(@class, "mega-nav-sandbox")]').remove # remove the hashicorp header
    doc.search('//a[text()="Edit this page"]').remove                # remove the 'edit this page' link

    File.open(filename, 'w') do |file|
      file.write(doc.to_html)
    end
  end
end

task :make_database do
  db = Sequel.connect('sqlite://docSet.dsidx', loggers: [Logger.new(STDOUT)])
  db.run('CREATE TABLE searchIndex(id INTEGER PRIMARY KEY, name TEXT, type TEXT, path TEXT);')
  db.run('CREATE UNIQUE INDEX anchor ON searchIndex (name, type, path);')

  Dir.chdir("#{URL}/docs/") do
    generate_entries(db, path: 'commands/*.html',         type: 'Command', title_sub: [/\n  »\n  /, /\n$/, ' Command'],        skip_file: 'index')
    generate_entries(db, path: 'provisioners/*.html',     type: 'Guide',   title_sub: [/\n  »\n  /, /\n$/, ' Provisioner'],    title_prefix: 'Provisioners:', skip_file: 'index')
    generate_entries(db, path: 'templates/*.html',        type: 'Guide',   title_sub: [/\n  »\n  /, /\n$/, 'Template', /\s+/], title_prefix: 'Templates:', skip_file: 'index')
    generate_entries(db, path: 'builders/*.html',         type: 'Guide',   title_sub: [/\n  »\n  /, /\n$/, ' Builder'],        title_prefix: 'Builders:', skip_file: 'index')
    generate_entries(db, path: 'post-processors/*.html',  type: 'Guide',   title_sub: [/\n  »\n  /, /\n$/, ' Post-Processor'], title_prefix: 'Post-Processors:', skip_file: 'index')
    generate_entries(db, path: 'extending/*.html',        type: 'Guide',   title_sub: [/\n  »\n  /, /\n$/, ' Extending'],      title_prefix: 'Extending:', skip_file: 'index')
  end

  # Add guides
  db[:searchIndex] << { name: 'Install Packer',           type: 'Guide', path: 'docs/install/index.html' }
  db[:searchIndex] << { name: 'Packer Terminology',       type: 'Guide', path: 'docs/basics/terminology.html' }
  db[:searchIndex] << { name: 'Command-Line',             type: 'Guide', path: 'docs/commands/index.html' }
  db[:searchIndex] << { name: 'Templates',                type: 'Guide', path: 'docs/templates/index.html' }
  db[:searchIndex] << { name: 'Builders',                 type: 'Guide', path: 'docs/builders/index.html' }
  db[:searchIndex] << { name: 'Provisioners',             type: 'Guide', path: 'docs/provisioners/index.html' }
  db[:searchIndex] << { name: 'Post-Processors',          type: 'Guide', path: 'docs/post-processors/index.html' }
  db[:searchIndex] << { name: 'Extending Packer',         type: 'Guide', path: 'docs/extending/index.html' }
  db[:searchIndex] << { name: 'Environment Variables',    type: 'Guide', path: 'docs/other/environment-variables.html' }
  db[:searchIndex] << { name: 'Core Configuration',       type: 'Guide', path: 'docs/other/core-configuration.html' }
  db[:searchIndex] << { name: 'Debugging',                type: 'Guide', path: 'docs/other/debugging.html' }

  db.disconnect
end

task :build_docset do
  contents_dir  = 'Packer.docset/Contents'
  resources_dir = "#{contents_dir}/Resources"
  documents_dir = "#{resources_dir}/Documents"

  system("mkdir -p #{documents_dir}")
  system("cp -r #{URL}/* #{documents_dir}/")
  system("cp Info.plist #{contents_dir}/")
  system("cp docSet.dsidx #{resources_dir}/")
  system('cp icon.png Packer.docset/')
end

task :compress_docset do
  system("tar --exclude='.DS_Store' -cvzf Packer.tgz Packer.docset")
end

def generate_entries(db, path:, type:, title_sub:, title_prefix: nil, skip_file: nil)
  Dir.glob(path).each do |filename|
    next if skip_file && filename.include?(skip_file)

    doc  = Nokogiri::HTML(File.open(filename))
    name = doc.css('h1').first.content
    name = title_sub.reduce(name) { |memo, sub_cmd| memo.sub(sub_cmd, '') } if title_sub
    name = [title_prefix, name].join(' ') if title_prefix

    entry = { name: name, type: type, path: File.join('docs', filename) }
    puts entry
    db[:searchIndex] << entry
  end
end
