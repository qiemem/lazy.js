def compile_file(output)
  require "closure-compiler"
  compiler = Closure::Compiler.new({
    :js_output_file => "#{output.chomp('.js')}.min.js",
    :externs        => File.join("lib", "externs.js"),
    :warning_level  => "VERBOSE"
  })

  puts compiler.compile_file(output)
end

# This is a simple hack to render single-line strings (or anyway, short text)
# from Markdown to HTML without wrapping in a <p> element.
def simple_markdown(text)
  @renderer ||= Redcarpet::Markdown.new(Redcarpet::Render::HTML)
  @renderer.render(text)[3...-3]
end

namespace :compile do
  desc "Compile lazy.js"
  task :lib do
    compile_file("lazy.js")
  end

  desc "Compile the homepage (currently hosted on GitHub pages)"
  task :site do
    require "json"
    require "mustache"
    require "nokogiri"
    require "pygments"
    require "redcarpet"

    markdown = File.read("README.md")

    #############################################
    # The README part
    #############################################

    # Translate to HTML w/ Redcarpet.
    renderer = Redcarpet::Markdown.new(Redcarpet::Render::HTML, :fenced_code_blocks => true)
    raw_html = renderer.render(markdown)

    # Parse HTML using Nokogiri.
    fragment = Nokogiri::HTML::fragment(raw_html)

    # Find the Travis build status icon and add GitHub and Twitter buttons.
    travis_image = fragment.css("a[href='https://travis-ci.org/dtao/lazy.js']").first
    travis_image.parent["class"] = "sharing"
    share_fragment = Nokogiri::HTML::fragment(File.read(File.join("site", "share.html")))
    travis_image.add_next_sibling(share_fragment)

    # Add IDs to section headings.
    fragment.css("h1,h2").each do |node|
      title = node.content
      node["id"] = title.downcase.gsub(/\s+/, "-").gsub(/[^\w\-]/, "")
    end

    # Add a distinguishing class to the 'Available functions' list so we can style it.
    fragment.css("#available-functions ~ ul").each do |node|
      node["class"] = "functions-list"
    end

    # Do syntax highlighting w/ Pygments.
    fragment.css("code").each do |node|
      language = node["class"]
      if language
        highlighted_html = Pygments.highlight(node.content, :lexer => language)
        replacement = Nokogiri::HTML::fragment(highlighted_html)
        node.parent.replace(replacement)
      end
    end

    #############################################
    # The API Docs part
    #############################################

    # OK so here's a hack: I'm going to strip out the first and last lines from
    # lazy.js so that JSDoc can read the annotations. (There is almost certainly
    # a more acceptable way to do this; but this is going to work so whatever.)
    #
    # Pretty awesome, right?
    File.open("temp.js", "w") do |f|
      f.write(File.read("lazy.js").lines[1..-2].join("\n"))
    end

    # Get a JSON representation of our JSDoc comments.
    classes = JSON.parse(`jsdoc temp.js --template templates/lazy`)

    # I called it temp.js for a reason, you guys!
    File.delete("temp.js")

    # OK, I want to finesse this data a little bit...
    classes.each_with_index do |class_data, index|
      class_data["description"] = simple_markdown(class_data["description"])

      # Hack, if you ask me (but this is how the Mustache docs do it).
      class_data["first"] = true if index == 0
      class_data["hide"] = true if index > 1

      class_data["methods"].each do |method_data|
        method_data["description"] = simple_markdown(method_data["description"])

        method_data["params"] && method_data["params"].each do |param_data|
          param_data["type"] = param_data["type"]["names"].join("|")
          param_data["description"] = simple_markdown(param_data["description"])
        end

        method_data["returns"] && method_data["returns"].each do |returns_data|
          returns_data["type"] = returns_data["type"]["names"].join("|")
          returns_data["description"] = simple_markdown(returns_data["description"])
        end
      end
    end

    #############################################
    # Putting it all together
    #############################################

    # Inject README into Mustache template.
    template = File.read("index.html.mustache")
    final_html = Mustache.render(template, {
      :readme  => fragment.inner_html,
      :classes => classes
    })

    # Finally, write the rendered result to index.html.
    File.open("index.html", "w") do |f|
      f.write(final_html)
    end
  end
end
