
#### 前言


源生成器的好处很多, 通过在编译时生成代码，可以减少运行时的反射和动态代码生成，从而提高应用程序的性能, 有时候需要对程序`AOT`以及裁剪编译的dll也是需要用SG来处理的。


我们开发程序应该都绕不过Mapper对象映射,用的比较多的库可能就是`AutoMapper`,`Maspter`之内的三方库吧;这些库很强大但是因为内部实现存在反射,因此开发的程序就没办法`AOT`了,因此如果程序不是很复杂但是又有很特殊的需求,建议使用SG来实现Mapper


#### 功能演示


这里我演示下自己开发的`AutoDto`生成DTO功能:


比如我们有一个User的类,需要生成UserDto



```
public class User
{
	public string Id { get; set; } = null!;
	public string FirstName { get; set; } = null!;
	public string LastName { get; set; } = null!;
	public int? Age { get; set; }
	public string? FullName => $"{FirstName} {LastName}";
}

```

定义UserDto并标注特性:



```
[AutoDto(nameof(User.Id))]//这里我们假设排除Id属性
public partial record UserDto;

```

就这样,源生成器将自动为我们生成对应的Dto:



```
partial record class UserDto
{
	/// 
	public string FirstName { get; set; }
	/// 
	public string LastName { get; set; }
	/// 
	public int? Age { get; set; }
}

```

并同时为我们生成一个简单的Mapper扩展方法:



```
public static partial class UserToUserDtoExtentions
{
	/// 
	/// mapper to UserDto
	/// 
	/// 
	public static UserDto MapperToUserDto(this User model)
	{
		return new UserDto()
		{
			FirstName = model.FirstName,
			LastName = model.LastName,
			Age = model.Age,
			FullName = model.FullName,
		};
	}
}

```

#### 实现代码



```
static void GENDTO(Compilation compilation, ImmutableArray nodes, SourceProductionContext context)
{
	if (nodes.Length == 0) return;
	StringBuilder envStringBuilder = new();
	envStringBuilder.AppendLine("// ");
	envStringBuilder.AppendLine("using System;");
	envStringBuilder.AppendLine("using System.Collections.Generic;");
	envStringBuilder.AppendLine("using System.Text;");
	envStringBuilder.AppendLine("using System.Threading.Tasks;");
	envStringBuilder.AppendLine("#pragma warning disable");

	foreach (var nodeSyntax in nodes.AsEnumerable())
	{
		//Cast()
		//Cast()
		if (nodeSyntax is not TypeDeclarationSyntax node)
		{
			continue;
		}
		//如果是Record类
		var isRecord = nodeSyntax is RecordDeclarationSyntax;
		//如果不含partial关键字,则不生成
		if (!node.Modifiers.Any(x => x.IsKind(SyntaxKind.PartialKeyword)))
		{
			continue;
		}

		AttributeSyntax? attributeSyntax = null;
		foreach (var attr in node.AttributeLists.AsEnumerable())
		{
			var attrName = attr.Attributes.FirstOrDefault()?.Name.ToString();
			if (attrName?.IndexOf(AttributeValueMetadataNameDto, System.StringComparison.Ordinal) == 0)
			{
				attributeSyntax = attr.Attributes.First(x => x.Name.ToString().IndexOf(AttributeValueMetadataNameDto, System.StringComparison.Ordinal) == 0);
				break;
			}
		}
		if (attributeSyntax == null)
		{
			continue;
		}
		//转译的Entity类名
		var entityName = string.Empty;
		string pattern = @"(?<=<)(?\w+)(?=>)";
		var match = Regex.Match(attributeSyntax.ToString(), pattern);
		if (match.Success)
		{
			entityName = match.Groups["type"].Value.Split(['.']).Last();
		}
		else
		{
			continue;
		}

		var sb = new StringBuilder();
		sb.AppendLine();
		sb.AppendLine($"//generate {entityName}-{node.Identifier.ValueText}");
		sb.AppendLine();
		sb.AppendLine("namespace $ni");
		sb.AppendLine("{");
		sb.AppendLine("$namespace");
		sb.AppendLine("$classes");
		sb.AppendLine("}");
		// sb.AppendLine("#pragma warning restore");
		string classTemp = $"partial $isRecord $className  {{ $body }}";
		classTemp = classTemp.Replace("$isRecord", isRecord ? "record class" : "class");

		{
			// 排除的属性
			List<string> excapes = [];

			if (attributeSyntax.ArgumentList != null)
			{
				for (var i = 0; i < attributeSyntax.ArgumentList!.Arguments.Count; i++)
				{
					var expressionSyntax = attributeSyntax.ArgumentList.Arguments[i].Expression;
					if (expressionSyntax.IsKind(SyntaxKind.InvocationExpression))
					{
						var name = (expressionSyntax as InvocationExpressionSyntax)!.ArgumentList.DescendantNodes().First().ToString();
						excapes.Add(name.Split(['.']).Last());
					}
					else if (expressionSyntax.IsKind(SyntaxKind.StringLiteralExpression))
					{
						var name = (expressionSyntax as LiteralExpressionSyntax)!.Token.ValueText;
						excapes.Add(name);
					}
				}
			}
			var className = node.Identifier.ValueText;
			var rootNamespace = string.Empty;
			//获取文件范围的命名空间
			var filescopeNamespace = node.AncestorsAndSelf().OfType().FirstOrDefault();
			if (filescopeNamespace != null)
			{
				rootNamespace = filescopeNamespace.Name.ToString();
			}
			else
			{
				rootNamespace = node.AncestorsAndSelf().OfType().Single().Name.ToString();
			}
			StringBuilder bodyBuilder = new();
			List<string> namespaces = [];
			StringBuilder bodyInnerBuilder = new();
			StringBuilder mapperBodyBuilder = new();
			bodyInnerBuilder.AppendLine();
			List<string> haveProps = [];
			// 生成属性
			void GenProperty(TypeSyntax @type)
			{
				var symbols = compilation.GetSymbolsWithName(@type.ToString(), SymbolFilter.Type);

				foreach (ITypeSymbol symbol in symbols.Cast())
				{
					var fullNameSpace = symbol.ContainingNamespace.ToDisplayString();
					// 命名空间
					if (!namespaces.Contains(fullNameSpace))
					{
						namespaces.Add(fullNameSpace);
					}
					symbol.GetMembers().OfType().ToList().ForEach(prop =>
																				   {
																					   if (!excapes.Contains(prop.Name))
																					   {
																						   // 如果存在同名属性,则不生成
																						   if (haveProps.Contains(prop.Name))
																						   {
																							   return;
																						   }

																						   haveProps.Add(prop.Name);

																						   //如果是泛型属性,则不生成
																						   if (prop.ContainingType.TypeParameters.Any(x => x.Name == prop.Type.Name))
																						   {
																							   return;
																						   }

																						   // prop:
																						   var raw = $"public {prop.Type.ToDisplayString()} {prop.Name} {{get;set;}}";
																						   // body:
																						   bodyInnerBuilder.AppendLine($"/// {@type}.{prop.Name}\" />");
																						   bodyInnerBuilder.AppendLine($"{raw}");

																						   // mapper:
																						   // 只有public的属性才能赋值
																						   if (prop.GetMethod?.DeclaredAccessibility == Accessibility.Public)
																						   {
																							   mapperBodyBuilder.AppendLine($"{prop.Name} = model.{prop.Name},");
																						   }
																					   }
																				   });
				}
			}

			// 生成属性:
			var symbols = compilation.GetSymbolsWithName(entityName, SymbolFilter.Type);
			var symbol = symbols.Cast().FirstOrDefault();
			//引用了其他库.
			if (symbol is null)
				continue;
GenProperty(SyntaxFactory.ParseTypeName(symbol.MetadataName));

			// 生成父类的属性:
			INamedTypeSymbol? baseType = symbol.BaseType;
			while (baseType != null)
			{
				GenProperty(SyntaxFactory.ParseTypeName(baseType.MetadataName));
				baseType = baseType.BaseType;
			}

			var rawClass = classTemp.Replace("$className", className);
			rawClass = rawClass.Replace("$body", bodyInnerBuilder.ToString());
			// append:
			bodyBuilder.AppendLine(rawClass);

			string rawNamespace = string.Empty;
			namespaces.ForEach(ns => rawNamespace += $"using {ns};\r\n");

			var source = sb.ToString();
			source = source.Replace("$namespace", rawNamespace);
			source = source.Replace("$classes", bodyBuilder.ToString());
			source = source.Replace("$ni", rootNamespace);

			// 生成Mapper
			var mapperSource = MapperTemplate.Replace("$namespace", namespaces.First());
			mapperSource = mapperSource.Replace("$ns", rootNamespace);
			mapperSource = mapperSource.Replace("$baseclass", entityName);
			mapperSource = mapperSource.Replace("$dtoclass", className);
			mapperSource = mapperSource.Replace("$body", mapperBodyBuilder.ToString());

			// 合并
			source = $"{source}\r\n{mapperSource}";
			envStringBuilder.AppendLine(source);
		}
	}

	envStringBuilder.AppendLine("#pragma warning restore");
	var envSource = envStringBuilder.ToString();
	// format:
	envSource = envSource.FormatContent();
	context.AddSource($"Biwen.AutoClassGenDtoG.g.cs", SourceText.From(envSource, Encoding.UTF8));
}

const string MapperTemplate = $@"
namespace $namespace
{{
    using $ns ;
    public static partial class $baseclassTo$dtoclassExtentions
    {{
        /// 
        /// mapper to $dtoclass
        /// 
        /// 
        public static $dtoclass MapperTo$dtoclass(this $baseclass model)
        {{
            return new $dtoclass()
            {{
                $body
            }};
        }}
    }}
}}
";

```

#### 最后


以上代码就完成了整个源生成步骤,最后你可以使用我发布的nuget包体验:



```
<ItemGroup>
   <PackageReference Include="Biwen.AutoClassGen.Attributes" Version="1.3.6" />
   <PackageReference Include="Biwen.AutoClassGen" Version="1.5.2" PrivateAssets="all" />
ItemGroup>

```

当然如果你对完整的实现感兴趣可以移步我的GitHub仓储,欢迎star [https://github.com/vipwan/Biwen.AutoClassGen](https://github.com)


 本博客参考[veee加速器](https://liuyunzhuge.com)。转载请注明出处！
